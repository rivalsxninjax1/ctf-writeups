# Beach Bar

**Platform:** TryHackMe
**Room:** Beach Bar
**Target IP:** 10.49.163.147 (hostname `tryhackme-2404`)
**Attacking Machine:** Kali Linux (`ninjax@ninjax`)


---

## 1. Overview

Beach Bar is a small Flask-based box themed around a beach bar's guest-facing "jukebox" app. Two weaknesses chain together into full compromise:

1. Leftover **demo credentials** exposed in an HTML comment on the login page.
2. **Unsafe YAML deserialization** (`yaml.load(Loader=yaml.Loader)`) in a playlist-import feature, giving remote code execution as a low-privileged user.
3. A **root-owned service leaking a secret via its process arguments**, which turned out to double as the root password — pure credential reuse.

No exploit-db CVE, no kernel bug this time — the whole box comes down to two very human mistakes: a demo account nobody disabled, and a "temporary" secret passed on a command line that never got cleaned up.

---

## 2. Reconnaissance

Standard opener — full port scan before deciding where to spend time:

```bash
nmap -Pn -p- -T4 --min-rate 2000 10.49.163.147
nmap -Pn -sV -T4 -p 22,80 10.49.163.147
```

**Result:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1
80/tcp open  http    (Flask / Gunicorn)
```

**Reasoning:** Only two doors. SSH advertises key-based auth only — password auth attempts return `Permission denied (publickey)` — so it's off the table as an entry point and gets mentally parked as a *destination* for later, same as always. That leaves port 80 as the only real attack surface, so everything starts there.

---

## 3. Web App Mapping

Hitting `/` returns a 302 to `/login` — a "Beach Bar // Sign in" page. Viewing source on that page is a reflex before touching the form itself, and it pays off immediately: an HTML comment left in the shipped page reads roughly "the demo DJ login is still enabled for the soft opening — swap this before the season starts," followed by the literal credentials `dj` / `dj`.

That's about as direct as a leak gets, so I authenticated with it:

```bash
curl -s -i -c cookies.txt -b cookies.txt -X POST http://10.49.163.147/login -d 'username=dj&password=dj'
```

This returns a 302 to `/dashboard` along with a Flask session cookie. Once authenticated, the app exposes:

- `/dashboard` — floor stats
- `/import` — paste or upload a YAML playlist
- `/export` — a sample YAML file to see the expected format
- `/logout`

`/import` is the interesting one — anywhere a web app accepts a structured file format from the user (YAML, XML, pickle, etc.) is worth a close look, because those parsers are a classic spot for developers to reach for the "full-featured" version of a library instead of the safe one.

---

## 4. Exploitation — YAML Deserialization to RCE

Submitting a harmless playlist through `/import` gets reflected back in the response as a Python `repr()` of the parsed object — not JSON, not a templated confirmation message, but literally the in-memory Python structure. Seeing a raw Python object printed back at you is a strong signal that the backend is using `yaml.load()` with the full `Loader` rather than `yaml.safe_load()`, because `safe_load` restricts what tags/types can be constructed and wouldn't produce output like that from arbitrary input.

PyYAML's unsafe loader supports `!!python/object/apply` tags, which let a YAML document instantiate and call arbitrary Python callables during parsing — not after some later "process the config" step, but during deserialization itself. That's the RCE primitive:

```yaml
!!python/object/apply :subprocess.check_output [["id"]]
```

Sent through `/import`, the app echoed back the command's output — code execution confirmed as a low-privileged `bartender` user before touching anything more invasive.

Since the app reflects raw command output, direct payloads with quotes/pipes tend to get mangled by YAML's own parsing rules, so the cleaner pattern is to base64-encode the actual shell command and decode-and-run it inside a nested `bash -c`:

```yaml
!!python/object/apply :subprocess.check_output [["bash","-c","echo <base64> | base64 -d | bash"]]
```

I wrapped that pattern into a small local helper so I wasn't hand-crafting YAML for every command — it base64-encodes whatever command I pass it, builds the payload, attaches the session cookie from `cookies.txt`, POSTs to `/import`, and pulls the output back out of the `<pre>` block in the response. From that point every subsequent command against the box goes through that helper instead of curl by hand.

---

## 5. Post-Exploitation — User Flag

With command execution established, the usual cheap-and-non-destructive commands first:

```bash
id; hostname; whoami; pwd; ls -la /home/bartender/
```

Confirmed `uid=1001(bartender)`, and a `user.txt` sitting in `/home/bartender/` with restrictive permissions readable by the app's own execution context. `cat`-ing it through the helper returns the **user flag**.

Worth noting for the privesc phase: the app and its virtualenv live under `/opt/beach-bar/`, and a second script — `jukeboxd.py` — is present and world-readable, tied to a systemd service that (per its filename and later confirmation) runs as root.

---

## 6. Privilege Escalation — Credential Reuse via a Root Process's Own Arguments

Rather than jumping to LinPEAS or a kernel-version check, the `jukeboxd.py` file sitting there world-readable next to a root-owned systemd service was too obvious a lead to ignore. Checking what's actually running:

```bash
ps auxf
```

turned up the root process invoked with a `--stream-pass` argument carrying what was clearly meant to be a secret — a real-looking password string passed in plaintext as a command-line flag. That's a mistake independent of what the flag is actually *for*: anything passed as a process argument is readable by any local user via `/proc/<pid>/cmdline`, so it's effectively as exposed as writing it to a world-readable file.

The natural next question is whether that "streaming password" was reused anywhere else. I ran it down a short checklist of every auth surface on the box:

- SSH as root/ubuntu/bartender → rejected (publickey-only, as seen in recon)
- `sudo -S -l` as bartender with that password → wrong password for bartender's own sudo
- `su - ubuntu` with that password → authentication failure
- `su - root` with that password → **success**, uid 0

`su` needs an interactive TTY to prompt for a password, which a plain reverse/webshell doesn't give you cleanly, so I used a small PTY wrapper script (`pty.fork()` + `select` to watch for the `Password:` prompt and write the password to the child's file descriptor at the right moment) to drive `su - root -c '<command>'` non-interactively through the existing RCE channel. Uploading and running that wrapper through the same base64-and-decode pattern used earlier confirmed `uid=0(root)` and pulled `root.txt` straight out of `/root/`.

**Root flag** obtained — the "streaming password" leaked in `jukeboxd`'s process arguments was, in fact, the root password.

---

## 7. Root Cause Summary

| # | Weakness | Where | Real-world equivalent |
|---|----------|-------|------------------------|
| 1 | Demo credentials shipped and left enabled | `dj` / `dj`, HTML comment on `/login` | Default/demo accounts never disabled before go-live |
| 2 | Unsafe YAML deserialization (`yaml.load(Loader=yaml.Loader)`) | `/import` playlist parser | OWASP — insecure deserialization |
| 3 | Hardcoded Flask `secret_key` | Application source | Predictable/static session signing key |
| 4 | Secret passed as a process command-line argument | `jukeboxd` service (`--stream-pass`) | Secrets exposed via `/proc/<pid>/cmdline` |
| 5 | Password reuse between an app-level secret and the root account | `su - root` | Credential reuse across trust boundaries |

### Suggested fixes (for a real environment)
- Always use `yaml.safe_load()` (or a schema-validating parser) for any YAML coming from user input — never the full `Loader`.
- Remove demo/default accounts before any deployment past local dev, and don't leave breadcrumbs about them in shipped HTML/JS comments.
- Never pass secrets as CLI arguments — use environment variables, a secrets manager, or a restricted config file instead, since process arguments are visible to any local user.
- Generate a random, per-deployment Flask `secret_key` rather than hardcoding one in source.
- Enforce unique credentials per account/service — a secret leaking in one place shouldn't grant access anywhere else.

---

## 8. Timeline / Methodology Recap

1. `nmap` full + version scan → SSH (22, key-only) and Flask/Gunicorn (80).
2. Viewed `/login` page source → demo credentials `dj`/`dj` leaked in an HTML comment.
3. Logged in → mapped authenticated surface (`/dashboard`, `/import`, `/export`, `/logout`).
4. Noticed `/import` reflecting raw Python objects → confirmed unsafe `yaml.load()` → built a `!!python/object/apply` RCE payload → wrapped it in a base64-decode-and-run helper script.
5. Confirmed code execution as `bartender` → read `user.txt`.
6. `ps auxf` → found a root process leaking a password via `--stream-pass`.
7. Checked that password against every local auth surface → worked for `su - root`.
8. Used a PTY-driving wrapper script to automate the interactive `su` prompt through the existing RCE channel → root shell → `root.txt`.

---

*End of write-up.*
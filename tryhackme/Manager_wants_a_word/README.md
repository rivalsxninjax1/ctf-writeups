# Management Wants a Word — TryHackMe Write-up

**Platform:** TryHackMe
**Event:** Hacker Holidays — Day 14
**Room:** Management Wants a Word
**Category:** Forensics
**Difficulty:** Hard
**Target artifact:** KAPE triage output from "Vera's" laptop (Room 214)

---

## 1. Overview

This room isn't an exploitation box at all — there's no shell to pop, no service to attack. It's a pure **digital forensics / DFIR** challenge built around a single scenario: a guest checked out early, IT pulled a full disk triage before wiping the machine, and somewhere in that triage sits a password the guest never meant to leave behind.

The "mindset" shift here is important. On a normal box I'm thinking like an attacker looking for a way *in*. Here I'm thinking like an investigator looking for a way *back* — reconstructing what the user (Vera) did, what Windows quietly remembered on her behalf, and following that chain until it unlocks something she deliberately hid. The in-room hint from `@0xMia` — *"a browser will remember things for you that you never told anyone else"* — is really the whole roadmap: this is a **Windows credential-recovery chain**, not a vulnerability chain.

So the plan going in was: extract the local password hash → use it to unwrap DPAPI-protected secrets → use DPAPI to unwrap Chrome's encryption key → use that key to decrypt a saved browser password → use that password to open an encrypted container. Each step only exists to unlock the next one.

---

## 2. Initial Triage

The evidence file downloaded from the room is a standard **KAPE** (Kroll Artifact Parser and Extractor) output — this is immediately recognizable from the directory layout rather than anything that needs guessing:

```
Users\Vera\...
Windows\System32\config\...
```

**Reasoning:** rather than opening files at random, I first mapped the two things a KAPE triage always contains that matter for a credential-recovery task:

- `Users\Vera\` — the user profile, which is where every user-specific artifact (browser data, DPAPI keys) will live.
- `Windows\System32\config\` — the registry hives (`SAM`, `SYSTEM`, `SECURITY`), which is where the raw password hash lives.

Identifying these two locations up front set the whole strategy: **hive first, then user profile**, because the hive is the thing that unlocks everything downstream.

![Screenshot](screenshots/Screenshot%202026-09-23%20at%2016.21.37.png)

---

## 3. Extracting the Local Password Hash (SAM + SYSTEM)

Windows never stores a plaintext password — it stores an NT hash, and that hash is exactly what's needed to derive DPAPI keys later. So the first real task was pulling it out of the offline hives.

I used Impacket's `secretsdump.py`, since it's built specifically for **offline** hive parsing (no live Windows box or admin session required — a hard requirement here, since this is a wiped machine and all we have are the raw hive files):

```bash
python3 secretsdump.py -sam SAM -system SYSTEM LOCAL
```

**Reasoning for this tool choice:** `-sam`/`-system LOCAL` mode is purpose-built for exactly this situation — dumping local account hashes from copied hive files rather than over a network. Mimikatz/Registry Explorer would work too, but secretsdump is the faster, scriptable path when I already have the hives sitting on disk.

**Result:** the NT hash for the local account `minivera`. This single hash is the key that everything else in this write-up depends on — it's what proves the user's identity to DPAPI without ever needing her actual password.

![Screenshot](screenshots/Screenshot%202026-09-23%20at%2016.25.17.png)

---

## 4. Unwrapping the DPAPI Master Key

This is the conceptual hinge of the whole room, so it's worth being explicit about *why* this step exists: Windows doesn't encrypt sensitive per-application data (browser passwords, Wi-Fi keys, etc.) with a fixed system key — it uses the **Data Protection API (DPAPI)**, where a "master key" is generated per user and is itself encrypted using material derived from that user's login credentials. Meaning: to decrypt anything DPAPI-protected, you first have to decrypt the master key, and to decrypt the master key you need the user's password (or, offline, their NT hash).

The master keys themselves live in a predictable, SID-scoped location:

```
Users\Vera\AppData\Roaming\Microsoft\Protect\<SID>\
```

**Reasoning for going here directly:** rather than searching the whole profile blindly, DPAPI master keys are always stored under `Protect\<SID>`, so this is a known, deterministic path — no fuzzing required, just knowledge of where Windows keeps this class of artifact.

Using the `minivera` NT hash recovered in Step 3, I decrypted the master key blob with a DPAPI-aware tool (DonPapi, though a forensic Python script using `dpapick`-style logic works identically). The output is a raw AES key that's valid for decrypting *any* DPAPI blob protected under Vera's profile — which is exactly what's needed next.

![Screenshot](screenshots/Screenshot%202026-09-23%20at%2016.26.02.png)

---

## 5. Recovering the Chrome AES Key from `Local State`

Modern Chrome (v10+) doesn't rely on DPAPI to encrypt saved passwords directly. Instead it generates its own **AES-256-GCM** key, and *that* key is what gets DPAPI-protected — a layer of indirection that's worth understanding rather than just following mechanically, since it explains why the master key from Step 4 isn't the final answer yet.

The wrapped key lives in a plain JSON config file:

```
Users\Vera\AppData\Local\Google\Chrome\User Data\Local State
```

Inside it, the relevant field is:

```
os_crypt.encrypted_key
```

**Reasoning:** this field is base64-encoded and prefixed with `DPAPI` before the actual ciphertext, which is the standard Chrome convention — so the extraction is really "strip the prefix, base64-decode, then decrypt using the DPAPI master key from Step 4." Once decrypted, the output is the **raw Chrome AES key** — the thing that actually protects individual saved passwords in the browser's database.

![Screenshot](screenshots/Screenshot%202026-09-23%20at%2016.27.15.png)

---

## 6. Decrypting the Saved Chrome Password

With the AES key in hand, the actual saved credential is sitting in Chrome's SQLite database:

```
Users\Vera\AppData\Local\Google\Chrome\User Data\Default\Login Data
```

I opened it with DB Browser for SQLite rather than trying to parse the file by hand — it's a standard SQLite database, so there's no reason to avoid standard tooling here. The relevant table is `logins`, and the relevant column is `password_value`, which holds a binary blob rather than plaintext.

**Reasoning for the decryption approach:** that blob follows Chrome's own encrypted-value format — a `v10`/`v11` prefix, a 12-byte nonce, the ciphertext, and a 16-byte GCM authentication tag appended at the end. Feeding those components into **AES-256-GCM** using the key from Step 5 recovers the plaintext password Vera had Chrome remember for her.

This is the "she never meant to leave it behind" moment from the room's briefing — the password was never typed anywhere an investigator could see it; it only existed because the browser silently retained it, exactly as `@0xMia`'s hint implied.

![Screenshot](screenshots/Screenshot%202026-09-23%20at%2019.05.13.png)

---

## 7. Locating and Mounting the Hidden Container

The final piece of the briefing — "it'll open a door to something she was keeping very quiet" — pointed at an encrypted container rather than another credential. Somewhere in the triage directory sits a **VeraCrypt volume**, and the classic trick for hiding one is exactly what was used here: a file with no extension, or one disguised as an ordinary document, so it blends in during casual browsing.

Once located, I mounted it on a Linux system rather than trying to do this in Windows — `cryptsetup` has native TCRYPT (TrueCrypt/VeraCrypt-compatible) support, so there's no need for the full VeraCrypt GUI just to read one container:

```bash
cryptsetup tcryptOpen <veracrypt_container_name> veracrypt_flag
```

When prompted, I entered the plaintext password recovered from Chrome in Step 6 — this is the point where every earlier step converges into a single action.

Then mounted the decrypted mapping:

```bash
mount /dev/mapper/veracrypt_flag /mnt/
```

![Screenshot](screenshots/Screenshot%202026-09-23%20at%2019.42.28.png)

---

## 8. Retrieving the Flag

Inside `/mnt/`, an invoice-styled document contained the objective of the whole chain.

```bash
ls /mnt/
cat /mnt/<file>
```

**Flag:**

```
THM{1t_w4s_V3r4_A11_Al0ng?!}
```

![Screenshot](screenshots/Screenshot%202026-09-23%20at%2019.53.36.png)
![Screenshot](screenshots/Screenshot%202026-09-23%20at%2019.58.38.png)

---

## 9. Root Cause Summary

| # | Weakness | Where | Real-world equivalent |
|---|----------|-------|------------------------|
| 1 | Local password hash recoverable from offline SAM/SYSTEM hives | Windows registry hives | Weak local account, no hive protection at rest |
| 2 | DPAPI master key decryptable once the account's NT hash is known | `AppData\Roaming\Microsoft\Protect\<SID>` | Credential-bound encryption is only as strong as the account's own secret |
| 3 | Browser AES key itself wrapped only by DPAPI, no additional secret | Chrome `Local State` | Single layer of protection for all saved-password material |
| 4 | Saved browser password recoverable via standard AES-GCM once the key is known | Chrome `Login Data` (SQLite) | Browser-based password storage as a high-value forensic target |
| 5 | Sensitive data hidden in a disguised encrypted container reachable via a browser-saved password | VeraCrypt container | Password reuse collapsing separate protection layers into one |

### Suggested fixes (for a real environment)
- Never let a single recovered secret (an NT hash) cascade into full compromise of every DPAPI-protected artifact on that account — full-disk encryption plus strong, unique account passwords narrows this chain significantly.
- Don't reuse a browser-saved password to protect a separate high-value secret (like an encrypted container) — password reuse is exactly what turned "recover a browser password" into "read confidential files."
- Treat browser credential stores as a primary forensic and attacker target, not an afterthought — enterprise policy should discourage saving high-value passwords in the browser at all.
- Disable or tightly control local password caching where possible, and monitor for extraction tooling (Mimikatz, secretsdump-style access to `SAM`/`SYSTEM`) in EDR telemetry.

---

## 10. Timeline / Methodology Recap

1. Extracted the KAPE triage archive → identified `Users\Vera\` and `Windows\System32\config\` as the two locations that matter.
2. `secretsdump.py -sam SAM -system SYSTEM LOCAL` → recovered the NT hash for `minivera`.
3. Located the DPAPI master key under `AppData\Roaming\Microsoft\Protect\<SID>\` → decrypted it using the NT hash.
4. Opened Chrome's `Local State` → extracted `os_crypt.encrypted_key` → decrypted it with the DPAPI master key to get the raw Chrome AES key.
5. Opened Chrome's `Login Data` SQLite database → extracted the `password_value` blob from `logins` → decrypted it with AES-256-GCM using the Chrome key → recovered Vera's plaintext saved password.
6. Located the disguised VeraCrypt container in the triage directory → mounted it with `cryptsetup tcryptOpen` using the recovered password.
7. Mounted the decrypted volume → read the invoice/text file inside → recovered the flag: `THM{1t_w4s_V3r4_A11_Al0ng?!}`

note: Screenshots are available below
![Screenshot](screenshots/Screenshot%202026-09-23%20at%2016.21.37.png)
![Screenshot](screenshots/Screenshot%202026-09-23%20at%2016.25.17.png)
![Screenshot](screenshots/Screenshot%202026-09-23%20at%2016.26.02.png)
![Screenshot](screenshots/Screenshot%202026-09-23%20at%2016.27.15.png)
![Screenshot](screenshots/Screenshot%202026-09-23%20at%2019.05.13.png)
![Screenshot](screenshots/Screenshot%202026-09-23%20at%2019.42.28.png)
![Screenshot](screenshots/Screenshot%202026-09-23%20at%2019.53.36.png)
![Screenshot](screenshots/Screenshot%202026-09-23%20at%2019.58.38.png)
![Screenshot](screenshots/Screenshot%202026-09-23%20at%2019.58.58.png)
![Screenshot](screenshots/Screenshot%202026-09-23%20at%2020.25.41.png)

---

*End of write-up.*
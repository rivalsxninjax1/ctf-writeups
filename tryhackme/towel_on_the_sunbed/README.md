# Towel on the Sunbed 

**Platform:** TryHackMe (Hacker Holidays — Byte Lotus Hotel)
**Room:** Towel on the Sunbed
**Attacking Machine:** Kali Linux (`ninjax@ninjax`) + Burp Suite


---

## 1. Overview

Towel on the Sunbed is a small logic-bug box, not a memory-corruption or injection one — the entire challenge is a single **race condition** in a "claim your reward" feature. The app hands out a currency called "ponzi" once every 24 hours, gated by a per-account cooldown timer. The vault needs 150 ponzi to open, and one legitimate claim gives nowhere near that. The whole point of the room is recognising that a cooldown enforced with a check-then-act pattern on the backend can be raced — if enough claim requests land at (effectively) the same instant, the server can be tricked into approving several of them before any single one has had the chance to update the "already claimed" state that the others are checking against.

No source code is given up front — the vulnerability has to be inferred from how the app *behaves* under normal use, then confirmed by trying to break the timing assumption directly.

---

## 2. Initial Recon — Understanding the Flow

Opening the target URL drops you on a login page, with a "Register" option to self-serve a new account rather than needing seeded credentials. After registering and logging in, the app lands on a portal with a single meaningful action: claim a reward.

The first claim works exactly as expected — a successful response, and a ponzi balance increases. Trying to claim again immediately gets rejected with an explicit message that a 24-hour timer is active. The vault itself states its requirement plainly: 150 ponzi to open.

**Reasoning:** one claim clearly doesn't get anywhere close to 150, and the timer is described as a hard 24-hour wait — so under intended usage this room would take weeks to solve one claim at a time. That mismatch (a huge currency requirement gated behind a slow-per-request mechanism) is usually the tell in these rooms: the "intended" bypass is to break the rate limit itself, not to grind it legitimately.

---

## 3. Confirming the Timer Is a Backend Check, Not Just a UI Lock

Before assuming a race condition, it's worth ruling out the boring case first — maybe the cooldown is purely cosmetic (a disabled button in the frontend) and the backend has no real check at all. I captured the claim request in Burp and replayed it as-is with Repeater.

**Result:** the server itself rejected the replayed request with the same "timer active" error. So the cooldown *is* enforced server-side, which rules out the trivial bypass (just re-hit the endpoint) and confirms this needs an actual timing attack against however that check is implemented.

---

## 4. The Race Condition — Theory

Cooldown checks like this are almost always implemented as two separate steps on the backend:

1. **Check:** has this session/account already claimed within the last 24 hours?
2. **Act:** if not, record a new claim and grant the reward.

If those two steps aren't wrapped in a single atomic operation (a proper row lock or an atomic increment), then multiple requests arriving close enough together can all pass step 1 — none of them see each other's "already claimed" write yet — before any of them get to step 2. The result is that a burst of near-simultaneous requests can each be individually approved, stacking up far more claims than the 24-hour limit was ever supposed to allow.

**The trick to actually pulling this off over HTTP:** normal sequential requests, even sent quickly, still complete one full round-trip at a time — plenty of opportunity for the server to update its state between them. What's needed instead is a way to have many requests arrive at the server *at essentially the same instant*, so they all hit the check step before any of them reach the act step. Burp Suite's Repeater has a feature built for exactly this: sending a group of requests in parallel with "last-byte sync," where Burp deliberately holds back the final byte of each request until every request in the group is ready, then releases them all at once — minimising the natural staggering that normal sequential or even rapid-fire requests would have.

---

## 5. Exploitation — Racing the Claim Endpoint

**Step 1 — Get a clean, unclaimed session.** Since the cooldown is tied to a session/account, the race needs to start from a session that hasn't claimed yet. I registered a fresh account, logged in, and captured the `connect.sid` session cookie for that fresh login.

**Step 2 — Capture and prep the claim request.** With the reward-claim request captured in Burp, I sent it to Repeater and swapped in the fresh session cookie from step 1, so the group of requests would all be racing against the same not-yet-claimed account state.

**Step 3 — Build a parallel request group.** In Repeater:
- Created a new request group.
- Duplicated the claim request roughly 99 times inside that group.
- Set the group's send mode to **"Send group in parallel (last-byte sync)"** — this is the setting that actually creates the race condition, by synchronising delivery of all 99 requests as tightly as possible.

**Step 4 — Fire the group.** Clicking "Send Group" fired all ~99 requests essentially simultaneously. Because they all arrived close enough together, a large batch of them landed in the window before the server had recorded the first claim — so instead of one claim succeeding and 98 being rejected by the timer, a large number of them succeeded outright.

**Step 5 — Confirm and open the vault.** Reloading the portal page showed the ponzi balance had jumped well past the 150 needed. Opening the vault at that point revealed the flag directly.

---

## 6. Root Cause Summary

| # | Weakness | Where | Real-world equivalent |
|---|----------|-------|------------------------|
| 1 | Time-of-check to time-of-use (TOCTOU) gap in the reward-claim logic | Reward-claim endpoint | Classic race condition — check and act not performed atomically |
| 2 | Rate/cooldown limiting enforced without a proper lock or atomic operation | 24-hour claim timer | Non-atomic "has user already done X" checks under concurrent load |
| 3 | No server-side detection of abnormal request bursts from a single session | Claim endpoint | Missing anomaly/rate-based defenses beyond the naive cooldown itself |

### Suggested fixes (for a real environment)
- Enforce the cooldown with an atomic database operation (e.g. a conditional update / `UPDATE ... WHERE last_claim < now() - interval` that only succeeds once) rather than a separate read-then-write.
- Use row-level locking or a transaction with proper isolation around the check-and-claim sequence so concurrent requests can't both pass the check before either writes the result.
- Add basic anomaly detection for a burst of near-identical requests from one session in a short window, independent of the cooldown logic itself.
- Treat any "limited action per time period" feature as needing the same concurrency rigor as a payment or inventory system — the underlying bug class is identical.

---

## 7. Timeline / Methodology Recap

1. Registered an account, logged in, explored the portal → found a single-purpose "claim reward" action and a vault requiring 150 ponzi.
2. Claimed once successfully, then found further claims blocked by an explicit 24-hour timer.
3. Replayed the claim request as-is in Burp Repeater to confirm the cooldown is enforced server-side, not just in the UI.
4. Reasoned through the check-then-act pattern most cooldown timers use and identified it as a likely race condition.
5. Registered a fresh account/session to get an unclaimed starting state.
6. Captured the claim request, swapped in the fresh session cookie, duplicated it ~99 times in a Repeater group, and sent it with "Send group in parallel (last-byte sync)."
7. A large batch of the parallel requests succeeded before the server-side state caught up, inflating the ponzi balance well past 150.
8. Reloaded the portal, confirmed the balance, opened the vault, and retrieved the flag.

---

*End of write-up.*
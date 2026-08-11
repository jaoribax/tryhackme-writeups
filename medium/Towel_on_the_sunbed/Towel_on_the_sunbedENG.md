# Whale Vault — TryHackMe

> **Difficulty:** Medium  
> **Category:** Web Exploitation · Business Logic Flaw · Race Condition (TOCTOU)  
> **Date:** 2025-08-08  
> **Room:** [https://tryhackme.com/room/whalevault](https://tryhackme.com/room/hh-towelonthesunbed-61271709)

---

## overview

Web application with a daily token reward system. The goal is to accumulate 150 tokens (the `whaleThreshold` parameter) to unlock the Whale Vault flag. The catch: each claim only grants 50 tokens, and the application's logic locks the account state after the first use. The solution isn't bypassing authentication — it's exploiting a **TOCTOU Race Condition (Time-of-Check / Time-of-Use)** to fire three simultaneous claims before the server can close access.

---

## reconnaissance

Initial access to the application at `http://<TARGET_IP>:3000`. A Guest account was created to interact with the interface.

Browser proxy configured to point to `127.0.0.1:8080` (Burp Suite).

Clicking **Claim Daily Reward** with Burp intercepting revealed that the claim is made via **POST method**, with data travelling in the request body — not exposed in the URL.

---

## enumeration

### Business logic analysis

With the POST packet held in Burp, the request anatomy was mapped:

- Each daily reward delivers **50 tokens**
- The `whaleThreshold` requires **150 tokens** to release the flag
- After the first claim, the account's logical state is **locked** — subsequent attempts return an error
- Bulk requests are blocked by rate limiting (**HTTP 429 Too Many Requests**)

A direct repeat attack doesn't work. The vector is different: exploit the window between the eligibility check and the state lock — **TOCTOU**.

---

## exploitation

### Race Condition — Last-Byte Sync

**Why TOCTOU works here:**

The application checks whether the account is eligible (Time-of-Check) and only then registers the use and locks the state (Time-of-Use). If three requests reach the server within the same time window — before any of them completes the check + lock cycle — the database attests eligibility for all three and releases 50 tokens for each concurrent process. Result: 150 tokens in a single round.

**Burp Suite setup:**

1. With the POST request intercepted, send to **Repeater** (`Ctrl+R`)
2. Duplicate the request tab until there are **exactly 3 identical tabs**
3. Select all 3 tabs → right-click → **Add to new tab group** → name it "Exploit Race"
4. In the group send menu, change the strategy to **Send group in parallel (last-byte sync)**
5. Fire the attack

**Why last-byte sync:**

Burp pre-establishes all TCP connections and sends the full content of all requests simultaneously — holding back only the final byte of each. By releasing all three final bytes at the same time, the requests arrive at the server processor in the same microcycle, forcing genuine concurrency.

**Why exactly 3 tabs:**

3 requests × 50 tokens = 150 tokens. Using more tabs would generate enough traffic to trigger the rate limiting (HTTP 429) — the minimum number needed is also the safe number.

**Result:**

All three tabs return **HTTP 200 OK**. Disabling intercept and refreshing the page in the application unlocks the Whale Vault.

> **Vulnerability:** Race Condition — TOCTOU (CWE-362)  
> **Root cause:** No atomic mechanism (lock or properly isolated transaction) between the eligibility check and the reward usage registration.  
> **MITRE ATT&CK:** T1499.004 — Application or System Exploitation

---

## flags

```
Flag: [captured in the Whale Vault after accumulating 150 tokens]
```

---

## attack chain

```
Application access → Guest account created
          │
          ▼
Burp Suite configured as proxy (127.0.0.1:8080)
          │
          ▼
Claim Daily Reward → POST intercepted
          │
          ▼
Analysis: 50 tokens/claim · 150 required · state locked after 1 use
          │
          ▼
Direct repeat attack blocked by rate limiting (HTTP 429)
          │
          ▼
Vector: TOCTOU — window between eligibility check and state lock
          │
          ▼
Burp Repeater → 3 identical tabs → Tab Group "Exploit Race"
          │
          ▼
Send group in parallel (last-byte sync)
          │
          ▼
3x HTTP 200 OK → 150 tokens accumulated
          │
          ▼
Whale Vault unlocked → flag captured
```

---

## what I learned

This room demonstrates a class of vulnerability that doesn't show up in automated scanners and has no CVE assigned — it's a **business logic flaw** exploited through precise timing.

The TOCTOU concept is simple in theory: there's a window between "check if allowed" and "register that it was used". If you can fit multiple operations into that window, the system authorizes all of them before closing access. The practical challenge is synchronizing requests with enough precision for that to actually happen.

Burp's **last-byte sync** solves this elegantly: by pre-establishing all connections and releasing all final bytes simultaneously, network delay is eliminated as a variable. The requests arrive at the server in the same microcycle — not approximately at the same time, but literally at the same time from the processor's perspective.

The rate limiting detail also mattered: using more than 3 tabs would have triggered the server's mitigation. Understanding the target's threshold before firing the attack was what made the exploitation surgical.

**Remediation:**

```sql
-- Operation must be atomic: check + register in a single transaction
BEGIN TRANSACTION;
  SELECT status FROM rewards WHERE user_id = ? FOR UPDATE;
  -- if eligible:
  UPDATE rewards SET status = 'used' WHERE user_id = ?;
  UPDATE wallets SET tokens = tokens + 50 WHERE user_id = ?;
COMMIT;
```

Without `FOR UPDATE` (or the ORM equivalent), the row is not locked during the transaction — allowing multiple concurrent reads to see the "eligible" state before any of them writes "used".

# Password Hashing

> **In one line —** storing passwords so that stealing your database does not hand an attacker everyone's password — using an algorithm deliberately designed to be slow.

| | |
|---|---|
| **Category** | Security Practice |
| **Architectural Layer** | Authentication |
| **Correct choices** | [Argon2id](Argon2.md) · [bcrypt](bcrypt.md) · scrypt |
| **Related notes** | [Authentication](Authentication.md) · [Argon2](Argon2.md) · [bcrypt](bcrypt.md) · [Secure Coding](Secure%20Coding.md) · [OWASP Top 10](OWASP%20Top%2010.md) |

---

## 1. Short Definition

*What is it?*

Password hashing converts a password into a fixed-length value that cannot be reversed, using a **deliberately slow, salted** algorithm designed specifically for this purpose.

---

## 2. Purpose

*What is its main purpose?*

To make a database breach survivable. Databases do get stolen; the question is whether the attacker walks away with usable passwords.

---

## 3. Problem

*What engineering problem does it solve?*

```text
PLAINTEXT           breach = every password, immediately
ENCRYPTED           breach = every password, once the key is found
                              (the key is usually in the same repo)
MD5 / SHA-256       breach = billions of guesses per second on a GPU
                              → common passwords cracked in seconds
Argon2 / bcrypt     breach = thousands of guesses per second
                              → strong passwords remain impractical to crack
```

> [!IMPORTANT]
> **Passwords must be hashed, never encrypted.** Encryption is reversible by design — that is the whole point of it. If you can recover the password, so can whoever takes your key.

---

## 4. Why general-purpose hashes are wrong

SHA-256 and MD5 were designed to be **fast**, because file integrity checking needs speed. That is precisely the wrong property here.

```text
SHA-256 on a GPU      ~10,000,000,000 guesses/second
bcrypt (cost 12)      ~   100–1,000 guesses/second
Argon2id (tuned)      ~   100–1,000 guesses/second, and memory-hard
```

> [!CAUTION]
> "We hash passwords with SHA-256" is a **vulnerability**, not a safeguard. Every real-world password dump cracked at scale — LinkedIn, Adobe, and many since — used a fast hash, sometimes without a salt.

---

## 5. Salt — and why it is not optional

A **salt** is a unique random value stored alongside each hash.

```text
WITHOUT A SALT                        WITH A SALT
same password → same hash             same password → different hashes
    ↓                                     ↓
rainbow tables work                   rainbow tables are useless
crack once, unlock every account      each password must be attacked separately
identical hashes reveal shared        no information leaks between accounts
passwords across users
```

> [!TIP]
> **You do not manage salts yourself.** Argon2 and bcrypt generate one per password and embed it in the output string. Handling salts manually is a sign something has gone wrong.

---

## 6. What a stored hash looks like

```text
$argon2id$v=19$m=65536,t=3,p=4$c29tZXNhbHQ$RdescudvJCsgt3ub+b+dWRWJTmaaJObG
 └──────┘      └─────────────┘ └─────────┘ └──────────────────────────────┘
 algorithm       parameters        salt                  hash
```

Everything needed to verify is in the string — including the parameters, which is what makes upgrading cost factors possible over time.

---

## 7. Which algorithm

| Algorithm | Verdict |
|---|---|
| **Argon2id** | **Preferred.** Winner of the Password Hashing Competition; memory-hard, so GPUs and ASICs gain far less |
| **bcrypt** | Still fine. Everywhere, battle-tested, 22 years old. Note its 72-byte input limit |
| **scrypt** | Fine; memory-hard, less common |
| **PBKDF2** | Acceptable only where required by compliance; weakest of the four |
| **SHA-*, MD5** | **Never.** Not for passwords |

See [Argon2](Argon2.md) and [bcrypt](bcrypt.md).

---

## 8. Tuning the cost

> [!IMPORTANT]
> The cost parameter should be set so that verification takes roughly **200–500 ms on your production hardware**. Slower is more secure and eventually annoys users and consumes CPU; faster helps attackers.

```text
Measure on YOUR hardware, not from a blog post
    ↓
Aim for ~250 ms
    ↓
Revisit every year or two — hardware gets faster, so cost must rise
```

Because the parameters are stored in the hash string, you can **upgrade on login**: verify with the old parameters, then re-hash with the new ones.

---

## 9. Real World Example

- **LinkedIn (2012)** — 6.5 million unsalted SHA-1 hashes; most were cracked within days.
- **Adobe (2013)** — encrypted rather than hashed, in ECB mode, with password hints. Effectively plaintext.
- **Well-handled breaches** exist too: services using bcrypt with a sensible cost have disclosed breaches where the passwords remained largely uncracked.

---

## 10. Verification is a comparison, never a decryption

```python
# registration
hash = argon2.hash(password)          # salt generated internally
store(hash)

# login
argon2.verify(stored_hash, password)  # re-hashes the input with the stored salt
```

> [!CAUTION]
> Comparison must be **constant-time**. A naive byte-by-byte comparison that exits on the first mismatch leaks information through timing. Every reputable library does this correctly — which is one more reason not to write your own.

---

## 11. Communication and Dependencies

- **A vetted library** — `argon2-cffi`, `bcrypt`, `passlib`, Spring Security, ASP.NET Identity
- **CPU and memory budget** — hashing is deliberately expensive
- **Rate limiting** — hashing slows attackers with your database; rate limiting slows attackers at your front door

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use a password hash for anything a human memorises and types: account passwords, PINs, recovery phrases.

> [!CAUTION]
> Do **not** use these for API keys and session tokens. Those are already high-entropy random values — a fast hash such as SHA-256 is appropriate there, because there is nothing to brute-force. Applying Argon2 to every API request would be a self-inflicted performance problem.

---

## 13. Advantages and Disadvantages

**Advantages**
- A stolen database does not yield usable passwords
- Salting defeats rainbow tables and hides password reuse
- Cost is tunable and upgradeable over time
- Mature, well-tested implementations in every language

**Disadvantages**
- Deliberately slow — real CPU and memory cost per login
- Poor parameters give false confidence
- Argon2's memory cost must be sized against your server
- Cannot help a user who chose `password123`

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Login latency** | 200–500 ms, by design |
| **CPU** | Significant per login — size your capacity for peak login rates |
| **Memory (Argon2)** | Configurable, e.g. 64 MB per hash — multiply by concurrency |
| **DoS risk** | An attacker flooding the login endpoint forces expensive hashing — **rate limit before hashing** |

> [!CAUTION]
> That last point is easy to miss: an unauthenticated endpoint that performs a 64 MB, 250 ms operation per request is a denial-of-service amplifier. Rate limit first, hash second.

---

## 15. Security Considerations

- **Never log passwords** — not in debug output, not in error reports, not in request logs
- **Enforce a minimum length (12+), not composition rules** — `P@ssw0rd!` satisfies most rules and is trivially cracked
- **Check against breached-password lists** (Have I Been Pwned) rather than adding more rules
- **Never truncate** — note bcrypt's 72-byte limit; pre-hashing with SHA-256 before bcrypt is the standard workaround
- **Invalidate all sessions** on password change
- **Rehash on login** when you raise the cost factor
- **A "pepper"** (a secret key stored outside the database) adds defence in depth, at the cost of key management complexity

---

## 16. Mental Model

> [!NOTE]
> **A password hash is a one-way meat grinder with an intentionally slow crank.**
>
> You can put a password in and compare the output, but nothing reconstructs the original. The slow crank is the point: it barely inconveniences one legitimate login and makes ten billion attacker guesses take centuries. The salt is a unique flavour added to each one, so grinding the same password for two users gives different results.

---

## 17. Mini Architecture Diagram

```text
Registration
password → [ Argon2id: salt + memory + iterations ] → stored hash string

Login
password + stored hash → same parameters re-applied → constant-time compare
    ↓
match / no match — the original is never recoverable
```

---

## 18. Complete Request Flow

```text
POST /login
    ↓
RATE LIMIT FIRST — before any hashing        ← DoS defence
    ↓
Look up the user; respond identically whether or not they exist
    ↓
argon2.verify(stored_hash, submitted_password)   ~250 ms
    ↓
No match → generic error, identical timing, attempt logged
    ↓
Match → is the stored hash using outdated parameters?
            → re-hash with current parameters and store
    ↓
Session created, ID regenerated
    ↓
─────────── breach scenario ───────────
Database stolen
    ↓
Attacker has salted Argon2id hashes
    ↓
~1,000 guesses/second per GPU instead of ~10 billion
    ↓
Strong passwords remain uncracked; users still need to be told
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> Hash passwords with Argon2id or bcrypt, tuned to ~250 ms — a fast hash such as SHA-256 is not a weaker choice, it is a broken one.

---

## 20. Common Mistakes

- **SHA-256 or MD5** for passwords
- **Encrypting instead of hashing**
- **No salt**, or a shared salt
- **Cost factor left at a library default from ten years ago**
- **Rolling your own** hashing or comparison
- **Rate limiting after hashing** instead of before
- **Composition rules** instead of length and breach checks
- **Logging passwords** in debug output

---

## 21. Open Source Technologies

- **argon2-cffi**, **bcrypt**, **passlib** — Python
- **bcrypt**, **argon2** — Node
- **Spring Security** `BCryptPasswordEncoder`, `Argon2PasswordEncoder`
- **ASP.NET Core Identity** — PBKDF2 by default; Argon2 available
- **Have I Been Pwned API** — breached password checks
- **OWASP Password Storage Cheat Sheet** — the reference for current parameters

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Find which algorithm and cost factor your application uses, and time one verification.
- [ ] Check whether rate limiting happens before or after password hashing on your login endpoint.
- [ ] Verify that your application re-hashes passwords on login when parameters change.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Password → Argon2id (salt + memory + time cost) → stored hash → constant-time verify
```

## 2. Request Flow

```text
Input       a submitted password and a stored hash string
    ↓
Processing  the stored parameters and salt are re-applied and compared
    ↓
Output      match or no match — never the original password
```

## 3. Real-World Usage

**The LinkedIn 2012 breach** — 6.5 million unsalted SHA-1 hashes — is still the clearest illustration. Most were cracked within days, and it changed industry practice more than any specification did.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Irreversible, salted, deliberately slow transformation of a password |
| **Why does it exist?** | Because databases are stolen and passwords are reused everywhere |
| **Where does it belong?** | At registration and at every login |
| **When should I use it?** | For every human-chosen secret — never for random API tokens |

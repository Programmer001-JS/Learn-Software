# bcrypt

> **In one line —** the password hash that has held up for twenty-five years: one tunable cost factor, available everywhere, and still a perfectly defensible choice.

| | |
|---|---|
| **Category** | Password Hashing Algorithm |
| **Architectural Layer** | Authentication |
| **Created** | 1999, based on the Blowfish cipher |
| **Related notes** | [Password Hashing](Password%20Hashing.md) · [Argon2](Argon2.md) · [Authentication](Authentication.md) |

---

## 1. Short Definition

*What is it?*

bcrypt is a password hashing function with a **configurable cost factor**. It generates its own salt, embeds everything needed for verification in the output string, and has been the industry default for two decades.

---

## 2. Purpose

*What is its main purpose?*

To make each password guess expensive, and to keep being expensive as hardware improves — by allowing the cost to be raised without changing the algorithm.

---

## 3. Problem

*What engineering problem does it solve?*

Hashes designed before bcrypt were fast and fixed. As hardware improved, they became crackable and there was no dial to turn.

```text
MD5, SHA-*      fixed speed → faster hardware makes them weaker every year
bcrypt          cost factor → raise it as hardware improves, forever
```

That adaptability is why an algorithm from 1999 is still in production use.

---

## 4. The cost factor

```text
cost = 10   →  2^10 = 1,024 iterations   ~50 ms
cost = 12   →  2^12 = 4,096 iterations   ~250 ms      ← current recommendation
cost = 14   →  2^14 = 16,384 iterations  ~1 s

Each +1 DOUBLES the work.
```

> [!TIP]
> **Cost 12 is the current sensible default**, but verify it on your own hardware — aim for roughly 250 ms. The right number rises over time, which is exactly the point of the parameter.

---

## 5. The stored format

```text
$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewKyOm2S/xVGXP4C
 └─┘ └┘ └──────────────────────┘└──────────────────────────┘
 alg cost         salt (22 chars)          hash (31 chars)
```

Version, cost and salt are all embedded. Verification needs nothing else, and outdated cost factors are detectable on login.

---

## 6. The 72-byte limit

> [!CAUTION]
> **bcrypt silently ignores everything after the first 72 bytes.** A 100-character passphrase is truncated, and the last 28 characters contribute nothing.

```text
"correct horse battery staple correct horse battery staple correct horse ba|ttery"
                                                                            ↑ ignored
```

This matters most for password managers generating long passphrases, and for non-ASCII passwords where a single character may be several bytes.

**The standard workaround:**

```python
# SHA-256 first (fixed 32-byte output), then bcrypt
bcrypt.hashpw(base64.b64encode(hashlib.sha256(password).digest()), salt)
```

> [!TIP]
> Base64-encode the SHA-256 output before passing it to bcrypt — a raw digest can contain a null byte, and some bcrypt implementations truncate at the first one. This detail has caused real vulnerabilities.

---

## 7. Architecture Position

```text
Registration / login
    ↓
Application
    ↓
bcrypt (cost 12, auto-generated salt)
    ↓
$2b$12$... stored in the database
```

---

## 8. bcrypt vs Argon2

| | bcrypt | [Argon2id](Argon2.md) |
|---|---|---|
| **Age / scrutiny** | 25 years | 10 years |
| **Memory-hard** | No (~4 KB) | **Yes**, configurable |
| **GPU resistance** | Moderate | Strong |
| **Input limit** | **72 bytes** | None |
| **Parameters** | One | Three |
| **Availability** | Universal, no native build issues | Good, sometimes needs compilation |

> [!IMPORTANT]
> **Argon2id is the better choice for new systems. bcrypt is not broken.** With cost 12 and unique salts, bcrypt remains genuinely strong — an existing deployment is not an incident, and migration can happen opportunistically on login rather than as an urgent project.

---

## 9. Real World Example

- **The default in Spring Security, ASP.NET Identity, Django (as an option), Rails and Laravel** — the widest deployment of any password hash.
- **Breaches where bcrypt held up** — several disclosed incidents involving bcrypt at a sensible cost reported that passwords were largely not cracked.
- **Breaches where it did not** — those used cost 4 or 5, which is barely slower than a plain hash.

---

## 10. Communication and Dependencies

- **A library** — `bcrypt` (Python, Node), `BCryptPasswordEncoder` (Spring), `BCrypt.Net`
- **CPU budget** — cost 12 is real work per login
- **Rate limiting**, applied before hashing

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use bcrypt when Argon2 is unavailable or awkward in your environment, when you need maximum library availability, or when you already have it deployed at a sensible cost.

> [!CAUTION]
> Do not use it for API keys or session tokens — those are random and high-entropy, and SHA-256 is the right tool. And do not use it for very long passphrases without the pre-hashing step above.

---

## 12. Advantages and Disadvantages

**Advantages**
- Twenty-five years of scrutiny with no practical break
- Available in every language, with no native build friction
- One parameter — hard to misconfigure badly
- Salt generated and embedded automatically
- Cost is upgradeable in place

**Disadvantages**
- **72-byte input truncation**, silently
- Not memory-hard — GPUs parallelise it more effectively than Argon2
- Only one tuning dimension
- Some old implementations have `$2a$` sign-extension quirks — prefer `$2b$`

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Time** | ~250 ms at cost 12 |
| **Memory** | ~4 KB — negligible, unlike Argon2 |
| **CPU** | Single-threaded per hash |
| **Concurrency** | CPU-bound; a login spike is a CPU spike |

> [!TIP]
> bcrypt's small memory footprint is an advantage in constrained containers, where Argon2's memory cost limits concurrency.

---

## 14. Security Considerations

- **Cost 12 or higher**, verified by timing on your hardware
- **Use `$2b$`**, not the legacy `$2a$`
- **Pre-hash long passwords** with SHA-256 and base64, to avoid truncation
- **Rate limit before hashing** — an unauthenticated 250 ms operation is a DoS vector
- **Re-hash on login** when you raise the cost factor
- **Never compare hashes manually** — use the library's constant-time verify
- **Never log passwords or hashes**

---

## 15. Mental Model

> [!NOTE]
> **bcrypt is a lock whose number of turns you can increase.**
>
> Each turn takes the same time for the owner as for a thief — but the owner turns it once and the thief must turn it a billion times. When lock-picking machines get faster, you add another turn rather than replacing the door.

---

## 16. Mini Architecture Diagram

```text
password (≤72 bytes, or SHA-256 pre-hashed)
    ↓
random salt generated
    ↓
bcrypt, 2^cost iterations
    ↓
$2b$12$salt+hash
    ↓
database
```

---

## 17. Complete Request Flow

```text
─────────── registration ───────────
password → bcrypt.hashpw(password, gensalt(12))
    ↓
salt generated, 4,096 iterations run, ~250 ms
    ↓
$2b$12$... stored

─────────── login ───────────
Rate limit checked FIRST
    ↓
Stored string parsed → cost and salt extracted
    ↓
Same computation re-run on the submitted password
    ↓
Constant-time comparison via bcrypt.checkpw()
    ↓
Cost below current standard? → re-hash at cost 12 and store

─────────── after a breach ───────────
Attacker holds salted bcrypt hashes at cost 12
    ↓
Roughly a few hundred guesses per second per GPU core, not billions
    ↓
Common passwords still fall; strong ones remain impractical
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> bcrypt at cost 12 with unique salts is still a strong choice — its one real trap is the silent 72-byte truncation, and Argon2id is the better default for new systems.

---

## 19. Common Mistakes

- **Low cost factors** (4–8) left from an old default or a test configuration
- **Ignoring the 72-byte limit** with long passphrases
- **Pre-hashing with a raw digest** containing a null byte, instead of base64
- **Using `$2a$`** where `$2b$` is available
- **Rate limiting after hashing**
- **Never raising the cost** as hardware improves
- **Manual hash comparison** instead of the library's verify

---

## 20. Open Source Technologies

- **bcrypt** libraries in Python, Node, Go, Ruby, PHP
- **Spring Security `BCryptPasswordEncoder`**
- **BCrypt.Net-Next** (.NET)
- **passlib** — supports bcrypt and Argon2, easing migration
- **OWASP Password Storage Cheat Sheet** — current cost recommendations

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Check the cost factor stored in your existing hashes — read the `$2b$NN$` prefix.
- [ ] Time a bcrypt verification at cost 12 on your production hardware.
- [ ] Test what your login does with a 100-character password.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Password → bcrypt (cost 12 + salt) → $2b$ string → database
```

## 2. Request Flow

```text
Input       a password (truncated at 72 bytes)
    ↓
Processing  2^cost iterations with the stored salt
    ↓
Output      a self-describing hash, verified in constant time
```

## 3. Real-World Usage

**bcrypt is the default in Spring Security, ASP.NET Identity, Laravel and Rails.** More production passwords are stored with it than with anything else — and the breaches where it failed were always cases of a very low cost factor, not of the algorithm.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A password hash with a tunable cost factor and built-in salting |
| **Why does it exist?** | Because fixed-speed hashes become weaker as hardware improves |
| **Where does it belong?** | Password storage |
| **When should I use it?** | When Argon2 is unavailable, or where it is already deployed at cost 12+ |

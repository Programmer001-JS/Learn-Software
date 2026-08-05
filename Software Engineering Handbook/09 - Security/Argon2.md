# Argon2

> **In one line —** the current best password hashing algorithm: deliberately slow *and* deliberately memory-hungry, which is what removes the attacker's GPU advantage.

| | |
|---|---|
| **Category** | Password Hashing Algorithm |
| **Architectural Layer** | Authentication |
| **Status** | Winner, Password Hashing Competition (2015) |
| **Variant to use** | **Argon2id** |
| **Related notes** | [Password Hashing](Password%20Hashing.md) · [bcrypt](bcrypt.md) · [Authentication](Authentication.md) |

---

## 1. Short Definition

*What is it?*

Argon2 is a **memory-hard** password hashing function. Beyond being computationally slow, it requires a configurable amount of RAM per hash, which makes massively parallel cracking hardware far less effective.

---

## 2. Purpose

*What is its main purpose?*

To close the gap that CPU-only slow hashes leave open: an attacker with thousands of GPU cores can parallelise computation cheaply, but cannot cheaply give every core 64 MB of dedicated memory.

---

## 3. Problem

*What engineering problem does it solve?*

```text
bcrypt      slow on CPU, uses ~4 KB of memory
    ↓
A GPU has thousands of cores and plenty of memory for 4 KB each
    ↓
Parallel cracking remains economical

Argon2id    slow AND requires (say) 64 MB per hash
    ↓
A GPU with 8 GB can run ~125 hashes in parallel, not 10,000
    ↓
The economics of large-scale cracking change substantially
```

---

## 4. The three variants

```text
Argon2d   maximum GPU resistance, but data-dependent memory access
          → vulnerable to side-channel timing attacks

Argon2i   side-channel resistant, weaker against GPU attacks

Argon2id  HYBRID — the first pass is data-independent, the rest data-dependent
          → resists both. This is the one to use.
```

> [!IMPORTANT]
> **Use Argon2id.** It is the variant recommended by OWASP and by RFC 9106. If a library defaults to Argon2i or Argon2d, set it explicitly.

---

## 5. The three parameters

```text
m  memory cost      how much RAM per hash        e.g. 64 MB
t  time cost        number of iterations         e.g. 3
p  parallelism      threads used                 e.g. 4
```

A reasonable current starting point (verify on your own hardware):

```text
m = 65536 KiB (64 MB) · t = 3 · p = 4     → roughly 200–400 ms
```

> [!TIP]
> **Tune to ~250 ms on your production hardware**, then raise memory before iterations — memory is what hurts the attacker most. And measure concurrency: 64 MB × 100 simultaneous logins is 6.4 GB of RAM.

---

## 6. The stored format

```text
$argon2id$v=19$m=65536,t=3,p=4$c29tZXNhbHQ$RdescudvJCsgt3ub+b+dWRWJTmaaJObG
 └──────┘ └──┘ └──────────────┘ └─────────┘ └────────────────────────────┘
 variant   ver     parameters       salt                  hash
```

Everything needed to verify — and to detect outdated parameters — is embedded. The salt is generated automatically; you never handle it.

---

## 7. Architecture Position

```text
Registration / login
    ↓
Application
    ↓
Argon2id: salt + memory + iterations
    ↓
Stored hash string in the database
```

---

## 8. Real World Example

- **The OWASP Password Storage Cheat Sheet** lists Argon2id as its first recommendation.
- **RFC 9106** standardised it, with parameter guidance.
- **1Password, Bitwarden and Signal** use Argon2 for key derivation from master passwords.
- **New projects** should default to it; existing bcrypt deployments are not urgent to migrate.

---

## 9. Argon2 vs bcrypt

| | Argon2id | [bcrypt](bcrypt.md) |
|---|---|---|
| **Memory-hard** | **Yes** — configurable | No (~4 KB fixed) |
| **GPU/ASIC resistance** | Strong | Moderate |
| **Age** | 2015 | 1999 |
| **Input length limit** | None | **72 bytes** |
| **Tuning** | Three parameters | One cost factor |
| **Availability** | Good, occasionally needs a native build | Everywhere |
| **Track record** | Shorter | 25 years of scrutiny |

> [!TIP]
> **Argon2id for new systems. bcrypt is not broken** — if you already run bcrypt with a sensible cost factor, that is a perfectly defensible position, and migration can happen opportunistically on login.

---

## 10. Communication and Dependencies

- **A library** — `argon2-cffi`, `argon2` (Node), `Argon2PasswordEncoder` (Spring), `Isopoh.Cryptography.Argon2` (.NET)
- **RAM budget** — memory cost multiplied by concurrent logins
- **Rate limiting**, applied before hashing

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Argon2id for password storage and for deriving encryption keys from user passphrases — both are cases where slowness is the feature.

> [!CAUTION]
> Never use it for API keys, session tokens or general hashing. Those are already high-entropy random values with nothing to brute-force; SHA-256 is correct there, and Argon2 would be a self-inflicted performance and memory problem on every request.

---

## 12. Advantages and Disadvantages

**Advantages**
- Memory-hardness substantially degrades GPU and ASIC attacks
- Three independent parameters allow precise tuning
- Modern design, standardised in RFC 9106
- No input length limit
- Parameters embedded in the hash, so upgrades are straightforward

**Disadvantages**
- Memory cost must be sized against real concurrency
- Some ecosystems need a native compilation step
- Fewer years of production scrutiny than bcrypt
- Three parameters are three opportunities to misconfigure
- Higher memory use is a denial-of-service consideration on login endpoints

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Time** | Target ~250 ms per verification |
| **Memory** | m × concurrent logins — the constraint people forget |
| **CPU** | Uses `p` threads per hash |
| **Container limits** | 64 MB per hash in a 512 MB container caps concurrency sharply |

> [!CAUTION]
> **Size the memory parameter against your container limit and expected concurrent logins.** Setting m = 256 MB on a small instance turns a login spike into an out-of-memory kill.

---

## 14. Security Considerations

- **Use Argon2id specifically**, not 2i or 2d
- **Follow current OWASP parameters** and revisit them periodically as hardware improves
- **Rate limit before hashing** — an unauthenticated 64 MB operation per request is a DoS amplifier
- **Re-hash on login** when you raise parameters
- **Never log the password**, and never log the hash either
- **Constant-time comparison** — handled by the library; do not compare strings yourself

---

## 15. Mental Model

> [!NOTE]
> **bcrypt makes each guess take a long time. Argon2 also makes each guess need a large desk.**
>
> An attacker can hire ten thousand people to guess in parallel. Giving each of them their own large desk is a much harder problem than giving each of them more time — and desk space is exactly what a GPU is short of.

---

## 16. Mini Architecture Diagram

```text
password
    ↓
random salt generated
    ↓
Argon2id  (m = 64 MB, t = 3, p = 4)
    ↓  fills and re-reads a large memory block
$argon2id$v=19$m=65536,t=3,p=4$salt$hash
    ↓
stored in the database
```

---

## 17. Complete Request Flow

```text
─────────── registration ───────────
password → argon2id.hash()
    ↓
salt generated internally; 64 MB allocated and worked over
    ↓
~250 ms → full hash string stored

─────────── login ───────────
Rate limit checked FIRST
    ↓
Stored string parsed → parameters and salt extracted
    ↓
The same computation re-run on the submitted password
    ↓
Constant-time comparison
    ↓
Parameters outdated? → re-hash with current settings and store

─────────── after a breach ───────────
Attacker has salted Argon2id hashes
    ↓
GPU parallelism limited by VRAM ÷ 64 MB
    ↓
Cracking rate collapses from billions to hundreds per second
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Argon2id is the current best choice for password storage because it is memory-hard — tune it to ~250 ms, and size the memory parameter against your real login concurrency.

---

## 19. Common Mistakes

- **Using Argon2i or Argon2d** instead of Argon2id
- **Default parameters** that are far too weak
- **Memory cost set without considering concurrency**, causing OOM under login spikes
- **No rate limiting** in front of an expensive hash
- **Migrating from bcrypt urgently** when bcrypt with a good cost factor is fine
- **Using it for API keys or session tokens**
- **Never revisiting parameters** as hardware improves

---

## 20. Open Source Technologies

- **argon2-cffi** (Python), **node-argon2**, **Spring `Argon2PasswordEncoder`**
- **libargon2** — the reference implementation
- **RFC 9106** — the specification and parameter guidance
- **OWASP Password Storage Cheat Sheet** — current recommended settings
- **passlib** — supports both Argon2 and bcrypt, easing migration

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Time an Argon2id hash on your production hardware and adjust parameters to reach ~250 ms.
- [ ] Multiply your memory cost by your expected peak concurrent logins and compare with your container limit.
- [ ] Check which variant your library uses by default.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Password → Argon2id (memory + time + parallelism) → hash string → database
```

## 2. Request Flow

```text
Input       a password
    ↓
Processing  salted, memory-hard derivation using stored parameters
    ↓
Output      a self-describing hash string, verifiable and upgradeable
```

## 3. Real-World Usage

**Password managers** such as Bitwarden and 1Password derive their vault keys with Argon2, because the master password is the single point of failure and memory-hardness is the strongest available defence against offline attack.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A memory-hard password hashing algorithm, in an id variant |
| **Why does it exist?** | Because CPU-only slowness no longer stops GPU cracking |
| **Where does it belong?** | Password storage and passphrase-based key derivation |
| **When should I use it?** | New systems — with parameters tuned to your hardware and concurrency |

# OWASP Top 10

> **In one line —** the ten categories of web vulnerability that actually cause breaches, ranked by real-world data rather than by how interesting they are.

| | |
|---|---|
| **Category** | Security Reference |
| **Architectural Layer** | Whole application |
| **Published by** | Open Worldwide Application Security Project |
| **Related notes** | [Secure Coding](Secure%20Coding.md) · [Authorization](Authorization.md) · [Authentication](Authentication.md) · [Secrets Management](Secrets%20Management.md) · [Validation](../07%20-%20Backend%20Design%20Patterns/Validation.md) |

---

## 1. Short Definition

*What is it?*

The OWASP Top 10 is a periodically updated list of the most critical web application security risks, compiled from data across hundreds of thousands of real applications.

---

## 2. Purpose

*What is its main purpose?*

To direct limited security attention at what actually goes wrong. Most teams cannot audit everything; this list says where to look first.

---

## 3. The list (2021 edition)

```text
A01  Broken Access Control              ← the number one, by a wide margin
A02  Cryptographic Failures
A03  Injection
A04  Insecure Design
A05  Security Misconfiguration
A06  Vulnerable and Outdated Components
A07  Identification and Authentication Failures
A08  Software and Data Integrity Failures
A09  Security Logging and Monitoring Failures
A10  Server-Side Request Forgery (SSRF)
```

> [!IMPORTANT]
> A separate **OWASP API Security Top 10** exists and is more relevant if you build APIs. Its number one is **Broken Object Level Authorization** — the same underlying problem as A01 here.

---

## 4. A01 — Broken Access Control

The most common and most damaging category.

```text
GET /api/orders/12345      → returns someone else's order
POST /api/users/42/role    → a normal user makes themselves admin
```

**Fix:** check authorisation on the **specific object**, in the [service layer](../07%20-%20Backend%20Design%20Patterns/Services.md), failing closed. See [Authorization](Authorization.md).

---

## 5. A02 — Cryptographic Failures

```text
✗ Passwords hashed with SHA-256 or stored encrypted
✗ Sensitive data transmitted over plain HTTP
✗ Hard-coded keys, or keys committed to git
✗ Home-made encryption
```

**Fix:** [Argon2id or bcrypt](Password%20Hashing.md) for passwords, TLS everywhere, keys in a [secrets manager](Secrets%20Management.md), and standard library crypto only.

---

## 6. A03 — Injection

SQL, NoSQL, OS command, LDAP — the same shape every time: **untrusted input becomes code**.

```python
f"SELECT * FROM users WHERE email = '{email}'"     # ✗
"SELECT * FROM users WHERE email = %s", (email,)   # ✓
```

**Fix:** parameterised queries always; ORMs do this by default. Note that **XSS** is now classified here too — the fix is escaping on output plus a Content Security Policy.

---

## 7. A04 — Insecure Design

New in 2021, and conceptually the most important addition: some vulnerabilities cannot be patched because the *design* is wrong.

```text
A password reset that emails the password back — no code change fixes that
No rate limiting anywhere in the design
A workflow where authorisation was never considered
```

**Fix:** threat-model before building. Ask "how would I abuse this?" at design time — see [Engineering Decision Making](../01%20-%20Foundation/03%20-%20Engineering%20Decision%20Making.md).

---

## 8. A05 — Security Misconfiguration

```text
✗ DEBUG = True in production          ← exposes environment variables and secrets
✗ Default credentials left in place
✗ Directory listing enabled
✗ Verbose error pages returning stack traces
✗ Cloud storage buckets left public
✗ Unnecessary ports and services exposed
```

**Fix:** harden by default, automate configuration, and scan for drift.

---

## 9. A06 — Vulnerable and Outdated Components

> [!CAUTION]
> **Log4Shell** is the defining example: a logging library used across the Java ecosystem allowed remote code execution from a logged string. Your dependencies are your attack surface, and most applications carry hundreds of transitive ones.

**Fix:** dependency scanning in CI (`npm audit`, Dependabot, Snyk, Trivy), a policy for patching, and an inventory of what you actually ship.

---

## 10. A07 — Identification and Authentication Failures

```text
✗ No rate limiting on login → credential stuffing succeeds
✗ Weak or reversible password storage
✗ Session IDs not regenerated after login
✗ No MFA option
✗ Predictable password reset tokens
```

**Fix:** see [Authentication](Authentication.md).

---

## 11. A08 — Software and Data Integrity Failures

```text
✗ CI/CD pipelines that trust unsigned artifacts
✗ Auto-updating from an unverified source
✗ Insecure deserialisation of untrusted data
```

**Fix:** sign artifacts, pin dependency versions with lockfiles, never deserialise untrusted input (`pickle`, Java serialisation, `BinaryFormatter`).

---

## 12. A09 — Logging and Monitoring Failures

> [!IMPORTANT]
> The average breach goes undetected for months. Not because the attack was invisible, but because nobody was looking — or because the evidence was never recorded.

```text
✓ Log authentication successes AND failures
✓ Log authorisation denials
✓ Log administrative actions
✗ NEVER log passwords, tokens, card numbers or full request bodies
✓ Alert on patterns: repeated 403s, login spikes, unusual data volumes
```

---

## 13. A10 — Server-Side Request Forgery

```text
User submits:  https://example.com/image.png    → fine
User submits:  http://169.254.169.254/latest/meta-data/iam/security-credentials/
    ↓
Your server fetches it — from inside your own network
    ↓
Cloud instance credentials returned to the attacker
```

> [!CAUTION]
> SSRF became a headline risk after major cloud breaches used exactly this path to reach the metadata service. Any feature that fetches a user-supplied URL — webhooks, image imports, link previews — is a candidate.

**Fix:** allow-list permitted hosts, block private IP ranges and the metadata endpoint, resolve DNS and validate the *resolved* address, and disable redirects.

---

## 14. When To Use / When NOT To Use

> [!TIP]
> Use the Top 10 as a **starting checklist** for reviews and as a training curriculum. Its greatest value is prioritisation: if you fix only A01, A03 and A06, you have addressed most real-world breach paths.

> [!CAUTION]
> Do not treat it as a complete security programme. It is a list of *categories*, not a specification, and "we checked the Top 10" is not the same as being secure. Business-logic flaws — abusing a discount, replaying a refund — appear nowhere on it.

---

## 15. Real World Example

- **Log4Shell (2021)** — A06, and the clearest demonstration that dependencies are your risk.
- **Capital One (2019)** — SSRF (A10) reaching cloud metadata credentials.
- **Almost every bug bounty report** on a public API is A01 in some form.

---

## 16. Mental Model

> [!NOTE]
> **The Top 10 is a list of the most common causes of house fires.**
>
> It does not describe every possible fire, and following it does not make a house fireproof. But if you only have time to check ten things, checking the ten that actually cause fires is a far better use of an afternoon than reading the full building code.

---

## 17. Mini Architecture Diagram

```text
Request
    ↓
A07 authentication         → is this really them?
    ↓
A01 authorisation          → may they do this, to THIS object?
    ↓
A03 input validation       → parameterised queries, escaped output
    ↓
Business logic  (A04 — was this designed with abuse in mind?)
    ↓
A02 crypto · A05 configuration · A06 dependencies · A10 outbound requests
    ↓
A09 logging and monitoring — throughout
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Broken access control is the number one real-world vulnerability — check the object, not just the role, and keep your dependencies patched.

---

## 19. Common Mistakes

- **Treating the list as complete** — business logic flaws are absent from it
- **Focusing on injection** while access control goes unchecked
- **Scanning for dependencies once** rather than continuously
- **Logging too much** — capturing tokens and passwords in log files
- **No alerting** on the events you do log
- **Assuming internal APIs are safe** — SSRF reaches them precisely because they are trusted

---

## 20. Open Source Technologies

- **OWASP ZAP** — dynamic application scanning
- **Semgrep**, **Bandit**, **CodeQL** — static analysis
- **Trivy**, **Grype**, **Dependabot**, **npm audit** — dependency scanning
- **OWASP Cheat Sheet Series** — the practical companion to the Top 10
- **OWASP ASVS** — a far more complete verification standard when you need one

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Take one API endpoint and try to access another user's data through it.
- [ ] Run a dependency scan on your project and count the known vulnerabilities.
- [ ] Find any feature that fetches a user-supplied URL and check it against the SSRF defences above.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Design (A04) → Auth (A07) → Access control (A01) → Input (A03)
    → Config (A05) · Crypto (A02) · Dependencies (A06) · Outbound (A10)
    → Logging (A09) across all of it
```

## 2. Request Flow

```text
Input       an untrusted request
    ↓
Processing  authenticated, authorised per object, validated, parameterised
    ↓
Output      a response that leaks nothing, with the event recorded
```

## 3. Real-World Usage

**Log4Shell** turned a logging library into a global emergency in December 2021. It remains the clearest argument that dependency management is security work, not maintenance work.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A data-driven ranking of the most critical web application risks |
| **Why does it exist?** | To direct limited security effort at what actually causes breaches |
| **Where does it belong?** | In code review, design review and developer training |
| **When should I use it?** | As a first checklist — never as a complete security programme |

# Authentication

> **In one line —** proving who someone is; distinct from what they may do, and the two are constantly confused.

| | |
|---|---|
| **Category** | Security Concept |
| **Architectural Layer** | Application boundary |
| **Abbreviation** | AuthN |
| **Related notes** | [Authorization](Authorization.md) · [Session Store](../08%20-%20Databases%20and%20Data/Session%20Store.md) · [JWT](JWT.md) · [OAuth2](OAuth2.md) · [Password Hashing](Password%20Hashing.md) |

---

## 1. Short Definition

*What is it?*

Authentication answers **"who are you?"** — verifying that a request comes from the identity it claims. [Authorization](Authorization.md) answers the separate question **"what may you do?"**.

```text
AuthN   who are you?      → 401 Unauthorized if it fails
AuthZ   what may you do?  → 403 Forbidden if it fails
```

> [!IMPORTANT]
> The HTTP status codes are named badly: `401 Unauthorized` actually means *unauthenticated*. Getting these two the right way round in your API is a small thing that makes client error handling possible.

---

## 2. Purpose

*What is its main purpose?*

To establish identity once per request so that every subsequent decision — authorisation, auditing, rate limiting, personalisation — has something to act on.

---

## 3. Problem

*What engineering problem does it solve?*

[HTTP is stateless](../04%20-%20Networking%20and%20Internet/HTTP%20HTTPS.md): every request arrives anonymous. Sending a password with each one would be unacceptable, so the system exchanges credentials once for a token that is presented afterwards.

```text
Login (credentials)  →  server verifies  →  issues a token
    ↓
Every later request carries the token, never the password
```

---

## 4. Architecture Position

```text
Client
    ↓  credentials or token
Middleware: AUTHENTICATION       ← you are here; identity established
    ↓
Middleware: AUTHORIZATION        ← may this identity do this?
    ↓
Controller → Service
    ↓
Object-level check: is this record theirs?
```

---

## 5. The three factors

```text
KNOWLEDGE   something you know      password, PIN
POSSESSION  something you have      phone, hardware key, TOTP app
INHERENCE   something you are       fingerprint, face
```

**Multi-factor authentication** requires two *different* categories. A password plus a security question is not MFA — both are knowledge.

> [!TIP]
> **SMS is the weakest second factor** — SIM swapping is a routine attack. TOTP apps are far better; **passkeys / WebAuthn** are better still, because they are phishing-resistant by design: the credential is bound to the domain and cannot be replayed on a lookalike site.

---

## 6. Token strategies

| Strategy | Identity lives | Revocation | Best for |
|---|---|---|---|
| **[Session](../08%20-%20Databases%20and%20Data/Session%20Store.md)** | Server-side, keyed by an opaque ID | **Instant** | Browser applications |
| **[JWT](JWT.md)** | Inside the signed token | Needs a blocklist | Multi-service, mobile |
| **API key** | A long-lived secret | Rotate or delete | Machine-to-machine |
| **[OAuth2](OAuth2.md) / [OIDC](OpenID%20Connect.md)** | Delegated to a provider | Provider-controlled | "Sign in with Google" |
| **mTLS** | A client certificate | Certificate revocation | Service-to-service |

---

## 7. The login flow

```text
POST /login  { email, password }
    ↓
Look up the user  — respond identically whether or not they exist
    ↓
Verify the password against the stored HASH  (never a plaintext comparison)
    ↓
Constant-time comparison
    ↓
MFA required? → second factor
    ↓
Issue a session or token; REGENERATE the session ID
    ↓
Set-Cookie: HttpOnly; Secure; SameSite
    ↓
Log the event: who, when, from where — successes AND failures
```

> [!CAUTION]
> **Never reveal whether an account exists.** "No such user" versus "wrong password" hands an attacker a list of valid accounts. Return one message, and make the timing identical too — otherwise the response time leaks the same information.

---

## 8. Real World Example

- **Passkeys** are replacing passwords across major platforms, because they cannot be phished, reused or leaked in a breach.
- **"Sign in with Google"** offloads credential storage entirely — you never hold a password, which removes a whole risk category.
- **Credential stuffing** is the dominant attack: passwords leaked from one site tried against another, which is why breach-password checks and MFA matter more than complexity rules.

---

## 9. Communication and Dependencies

- **[Password hashing](Password%20Hashing.md)** — Argon2 or bcrypt, never a general-purpose hash
- **A session store or signing key**
- **TLS** — credentials over plaintext are compromised in transit
- **Rate limiting** — the primary defence against brute force
- **An identity provider**, if delegating

---

## 10. When To Use / When NOT To Use

> [!TIP]
> **Delegate authentication when you reasonably can.** OAuth2/OIDC via Google, Microsoft or a managed identity provider means you never store passwords — and cannot leak them. For most products this is the correct default.

> [!CAUTION]
> Do not implement your own authentication protocol, do not invent your own token format, and do not write your own password hashing. Every one of these has well-tested libraries, and every custom implementation has failed in the same well-documented ways.

---

## 11. Advantages and Disadvantages

**Passwords**
- Universal and understood; no dependencies
- ✗ Reused, phishable, leaked in breaches, weak by default

**Sessions**
- Instant revocation, small cookie, simple to reason about
- ✗ Requires a shared store

**JWT**
- Stateless verification, works across services
- ✗ Revocation is genuinely hard

**Passkeys**
- Phishing-resistant, nothing to leak
- ✗ Device recovery, and still not universally supported

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Password hashing** | Deliberately slow — 100–500 ms is correct, not a bug |
| **Session lookup** | One cache read per request |
| **JWT verification** | Signature check only; no lookup |
| **MFA** | User-perceived latency, not server cost |

> [!IMPORTANT]
> Password hashing being slow is the **point**. A fast hash means an attacker with your database can test billions of guesses per second. Never "optimise" it — see [Password Hashing](Password%20Hashing.md).

---

## 13. Security Considerations

> [!CAUTION]
> Authentication is the highest-value target in any application. These are not optional refinements — each one has been the root cause of real breaches.

- **Hash passwords with Argon2id or bcrypt** — never SHA-256, never MD5, never encryption
- **Rate-limit login attempts** by IP *and* by account; lock out or add delay after repeated failures
- **Regenerate the session ID on login** — otherwise session fixation
- **Cookie flags**: `HttpOnly`, `Secure`, `SameSite`
- **Constant-time comparison** for tokens and hashes
- **Uniform responses and timing** for unknown accounts
- **Invalidate all sessions on password change**
- **Secure the password reset flow** — single-use, short-lived, unpredictable tokens; this is frequently the weakest link
- **Log authentication events** — you cannot investigate what you did not record
- **Check passwords against known-breach lists** rather than enforcing complexity rules that produce `Password1!`

---

## 14. Mental Model

> [!NOTE]
> **Authentication is showing your passport at the border; authorisation is the visa that says what you may do inside.**
>
> The border officer verifies you are who you claim. Whether you may work, study or only visit is a separate document entirely — and confusing the two is exactly the bug where an application checks that someone is logged in and never checks whether the record belongs to them.

---

## 15. Mini Architecture Diagram

```text
Client
    ↓
TLS
    ↓
Rate limiter          → 429
    ↓
AUTHENTICATION        → 401 if identity cannot be established
    ↓
identity attached to the request
    ↓
AUTHORIZATION         → 403
    ↓
Handler → object-level ownership check
```

---

## 16. Complete Request Flow

```text
─────────── login ───────────
POST /login
    ↓
Rate limit check                  → 429 after repeated failures
    ↓
User lookup — identical response and timing whether found or not
    ↓
Argon2 verify (~200 ms, deliberately)
    ↓
MFA challenge, if enabled
    ↓
Session created, ID regenerated, cookie set with all three flags
    ↓
Event logged

─────────── every later request ───────────
Cookie or Authorization header presented
    ↓
Session looked up, or token signature verified
    ↓
Expired or missing → 401
    ↓
Identity attached → authorisation decides the rest
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> Authentication establishes identity and nothing more — it must be slow to brute-force, revocable, and always followed by a separate authorisation decision.

---

## 18. Common Mistakes

- **Confusing authentication with authorisation** — the most consequential mistake in this area
- **Revealing whether an account exists**, through the message or the timing
- **No rate limiting** on login
- **Fast hashing** — or worse, storing passwords reversibly
- **Not regenerating the session ID** after login
- **Weak password reset flows** — long-lived, guessable or reusable tokens
- **Rolling your own** crypto, tokens or protocol
- **SMS as the only second factor**

---

## 19. Open Source Technologies

- **Argon2**, **bcrypt** — password hashing
- **Keycloak**, **Ory**, **Authentik**, **SuperTokens** — self-hosted identity providers
- **Auth0**, **Clerk**, **Firebase Auth** — managed alternatives
- **passlib**, **Spring Security**, **ASP.NET Identity**, **Passport.js**
- **WebAuthn / passkey** libraries
- **Have I Been Pwned API** — check passwords against known breaches

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Check whether your login endpoint is rate-limited by account as well as by IP.
- [ ] Time the response for a valid versus an unknown email address. Do they differ?
- [ ] Review your password reset flow: are tokens single-use, short-lived and unguessable?

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Client → TLS → rate limit → authentication → authorisation → handler
```

## 2. Request Flow

```text
Input       credentials, or a token from a previous authentication
    ↓
Processing  verified against a stored hash or a signature; identity established
    ↓
Output      an authenticated request — or 401, revealing nothing extra
```

## 3. Real-World Usage

**Credential stuffing** — reusing passwords leaked elsewhere — is the most common successful attack on login systems. It is why MFA and breach-password checks protect users far more effectively than complexity requirements ever did.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Verifying the claimed identity behind a request |
| **Why does it exist?** | Because HTTP requests arrive anonymous and untrusted |
| **Where does it belong?** | At the boundary, before any authorisation decision |
| **When should I use it?** | Whenever identity matters — delegating it where you reasonably can |

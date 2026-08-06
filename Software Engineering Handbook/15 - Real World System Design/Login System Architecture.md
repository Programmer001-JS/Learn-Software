# Login System Architecture

> **In one line —** the system every application needs and almost nobody should build from scratch; and if you do build it, the hard parts are session lifecycle and account recovery, not password checking.

| | |
|---|---|
| **Category** | Case Study *(system design)* |
| **Architectural Layer** | Application / Security |
| **Related notes** | [Authentication](../09%20-%20Security/Authentication.md) · [Authorization](../09%20-%20Security/Authorization.md) · [JWT](../09%20-%20Security/JWT.md) · [OAuth](../09%20-%20Security/OAuth.md) · [Password Hashing](../09%20-%20Security/Password%20Hashing.md) · [Redis](../08%20-%20Databases%20and%20Data/Redis.md) |

---

## 1. The System in One Page

*What are we designing?*

```text
A user proves who they are, and then stays proven for a while.

    REGISTER    create an identity
    LOGIN       verify a credential → issue a session
    AUTHORISE   check the session on every subsequent request
    REFRESH     extend a session without re-authenticating
    LOGOUT      end the session — everywhere, if asked
    RECOVER     regain access without the credential
```

> [!IMPORTANT]
> **Password verification is the easy 10% of this system.** The parts that go wrong in production are session revocation, refresh token rotation, and account recovery — because those are where an attacker attacks and where the edge cases live. If you are designing this, spend your time there.

---

## 2. Requirements

```text
FUNCTIONAL
    email + password, and social sign-in (OAuth)
    multi-factor authentication
    "remember me" across devices
    log out of one device, or all devices
    password reset that cannot be abused to take over accounts
    account lockout that cannot be used to lock out others

NON-FUNCTIONAL
    authorisation check on EVERY request      → must be ~1 ms
    login itself may take 200 ms              → hashing is deliberately slow
    availability: if login is down, the product is down
    audit trail of every authentication event
```

---

## 3. The build-versus-buy decision, first

```text
BUY / DELEGATE                       BUILD
Auth0, Clerk, Cognito, Keycloak      your own tables and endpoints
MFA, social login, SSO included      you implement each one
security patched by specialists      you own every CVE
per-user cost, vendor lock-in        full control, no per-user fee
migration is painful later           you carry breach liability
```

> [!TIP]
> **Delegate authentication unless you have a specific reason not to.** Authentication is a solved problem with severe consequences for getting it wrong, and it is not where products differentiate. The legitimate reasons to build are regulatory data residency, extreme cost at very large user counts, or genuinely unusual requirements. "We can do it ourselves" is true and beside the point. Note that **authorisation** — what a user may do — is domain logic you should almost always own; see [Authorization](../09%20-%20Security/Authorization.md).

---

## 4. Architecture

```text
                            client
                              │
                    ┌─────────▼──────────┐
                    │   API gateway /     │  rate limits login attempts
                    │   load balancer     │  ← the FIRST defence
                    └─────────┬──────────┘
                              │
        ┌─────────────────────▼─────────────────────┐
        │            AUTH SERVICE                    │
        │  ┌──────────────────────────────────────┐ │
        │  │ /register  /login  /refresh  /logout │ │
        │  │ /mfa  /password-reset  /oauth        │ │
        │  └──────────────────────────────────────┘ │
        └───┬──────────────┬───────────────┬────────┘
            ▼              ▼               ▼
     ┌────────────┐ ┌─────────────┐ ┌──────────────┐
     │ users DB   │ │ SESSION     │ │ email / SMS  │
     │ id, email  │ │ STORE       │ │ (async, via  │
     │ argon2 hash│ │ (Redis)     │ │  a queue)    │
     │ mfa secret │ │ refresh     │ └──────────────┘
     │ created_at │ │ tokens,     │
     └────────────┘ │ revocations │
                    └─────────────┘
            │
     ┌──────▼──────────────────┐
     │ audit log (append-only) │  every auth event, forever
     └─────────────────────────┘
```

---

## 5. Passwords: the parts that are non-negotiable

```text
STORE            argon2id (preferred), scrypt, or bcrypt
                 NEVER: MD5, SHA-1, SHA-256, "encrypted", plaintext
                 → a fast hash is the wrong tool; slowness is the feature

SALT             automatic in all three; per-password, never global

COST             tune so hashing takes ~100-300 ms on YOUR hardware
                 → and revisit it every couple of years

VALIDATE         length ≥ 12, and check against a breached-password list
                 → complexity rules produce Password1! and nothing else

NEVER            log it, email it, put it in a URL, or truncate it silently
```

> [!CAUTION]
> **Login must take the same amount of time whether the email exists or not.** If a missing user returns in 5 ms and an existing one in 200 ms, you have built an account enumeration oracle. Hash a dummy value on the not-found path. The same applies to messages: "invalid email or password" for both cases, and identical behaviour on password reset for unknown addresses.

---

## 6. Sessions: the design decision that matters most

```text
OPAQUE SESSION TOKEN (server-side state)          STATELESS JWT
random id in a cookie; state in Redis             signed claims in the token
    ✓ REVOCATION IS INSTANT                           ✗ valid until it expires
    ✓ small, nothing leaks                            ✓ no lookup per request
    ✗ a store lookup per request (~1 ms in Redis)     ✓ scales trivially
    → the right default for a web application         → good for short-lived
                                                        service-to-service use
```

```text
THE PRACTICAL COMPROMISE
    short-lived ACCESS token (5-15 min, stateless or opaque)
    long-lived REFRESH token (days-weeks, ALWAYS server-side and revocable)
    → revocation is bounded by the access token's lifetime
```

> [!IMPORTANT]
> **The question that decides this is "how fast must logout work?"** If a compromised session must die immediately — and for anything handling money, health data or admin capability it must — you need server-side state, or a revocation list, which is server-side state wearing a different hat. Long-lived stateless JWTs as session tokens are the single most common serious mistake in modern authentication design.

---

## 7. Refresh token rotation, and detecting theft

```text
EVERY refresh issues a NEW refresh token and invalidates the old one.
    ↓
If an OLD refresh token is presented again:
    → either the legitimate client retried, or a token was STOLEN
    → treat it as theft: revoke the whole token family, force re-login
```

```text
Store refresh tokens hashed, with:
    a family id, a device fingerprint, the issue time, last-used time
    → which powers "log out other devices" and "your active sessions"
```

> [!TIP]
> **Rotation with reuse detection is what turns a stolen refresh token from a permanent backdoor into a detectable, self-limiting event.** It is perhaps 30 lines of logic and it is the difference between a session system that fails safely and one that does not.

---

## 8. Cookies versus headers

```text
COOKIES (recommended for browsers)
    HttpOnly            JavaScript cannot read it → XSS cannot steal it
    Secure              HTTPS only
    SameSite=Lax/Strict CSRF protection
    __Host- prefix      binds it to the exact host
    ↓
    plus CSRF protection for state-changing requests

AUTHORIZATION HEADER (for native apps and APIs)
    must be stored somewhere → localStorage is READABLE BY XSS
    no CSRF exposure, because it is not sent automatically
```

> [!CAUTION]
> **Storing a session token in `localStorage` means any cross-site scripting vulnerability is a full account takeover.** The common argument — "JWTs in localStorage avoid CSRF" — trades a well-understood, easily-mitigated risk for a much worse one. For browser applications: HttpOnly cookies plus CSRF protection. This is not a close call.

---

## 9. Multi-factor authentication

```text
STRENGTH, weakest to strongest
    SMS codes         better than nothing; vulnerable to SIM swapping
    TOTP (authenticator app)   good, offline, the sensible default
    push approval     good UX, vulnerable to approval fatigue
    WebAuthn / passkeys        PHISHING-RESISTANT — the real answer
```

```text
DESIGN DETAILS PEOPLE MISS
    recovery codes, generated once, shown once, hashed at rest
    MFA must be enforced on password RESET too — or it is bypassable
    rate limit code attempts (6 digits = 1,000,000 guesses)
    a "remember this device" token, so MFA is not every login
```

> [!TIP]
> **Passkeys (WebAuthn) are the first authentication method that structurally defeats phishing**, because the credential is bound to the origin and never leaves the device. Where you can offer them, do — and note that they replace the password rather than supplementing it, which simplifies the whole flow.

---

## 10. Password reset: the most attacked flow

```text
CORRECT
    always respond identically, whether or not the address exists
    generate a HIGH-ENTROPY token; store only its HASH
    single use, expires in 15-60 minutes
    INVALIDATE ALL SESSIONS on successful reset
    require MFA if enabled
    email the user that a reset occurred — to the OLD address too

WRONG
    a token derived from user id or timestamp    → forgeable
    reusable, or valid for days
    "no account with that email"                 → enumeration
    sessions left alive                          → the attacker keeps access
    email change without confirming BOTH addresses → silent takeover
```

> [!CAUTION]
> **Password reset is functionally a second authentication mechanism, and it is usually the weakest one.** An attacker does not attack your argon2 parameters; they attack the reset flow, the email change flow, or the support process that resets accounts on request. Design all three with the same care as login itself — including the human one.

---

## 11. Rate limiting and lockout

```text
LIMIT BY SEVERAL DIMENSIONS
    per account      → stops brute force on one user
    per IP           → stops one source attacking many users
    global on /login → catches distributed credential stuffing

LOCKOUT IS A TRAP
    hard lockout after N failures = anyone can lock out any user
    → prefer exponential backoff, and CAPTCHA on suspicion
    → notify the user, do not silently deny them
```

> [!IMPORTANT]
> **Credential stuffing — replaying passwords leaked from other sites — is the dominant real attack, not brute force.** Per-account limits do not stop it, because each account sees only one or two attempts. The defences that work are checking passwords against breach corpora at registration, MFA, and anomaly detection on the login endpoint as a whole.

---

## 12. Real World Example

- **Google, GitHub, Microsoft** — passkeys plus opaque server-side sessions, with per-device session management visible to the user.
- **Any application using "Sign in with Google"** — delegating both credential handling and MFA; see [OAuth](../09%20-%20Security/OAuth.md).
- **Banking applications** — short access tokens, aggressive re-authentication for sensitive actions, full audit trails.
- **Enterprise SSO** — SAML or OIDC against a corporate identity provider, with the application holding no credentials at all.
- **Keycloak or Authentik**, self-hosted, where data residency rules out a SaaS provider.

---

## 13. Communication and Dependencies

- **A session store** — Redis, with persistence and replication; it is now a critical dependency
- **A relational database** for users, with unique constraints and an audit table
- **Email and SMS delivery**, asynchronously via a queue — never blocking the request
- **Rate limiting at the edge**; see [Load Balancing](../14%20-%20Scalability%20and%20Reliability/Load%20Balancing.md)
- **A breached-password list** (Have I Been Pwned's k-anonymity API, or a local copy)
- **Secrets management** for signing keys, with rotation; see [Secrets Management](../09%20-%20Security/Secrets%20Management.md)
- **Monitoring on authentication failure rates**, which is both a security and an availability signal

---

## 14. When To Use / When NOT To Use

> [!TIP]
> Build this yourself only when a provider genuinely cannot meet a requirement. If you do, use a well-maintained framework library rather than writing the primitives — the library has had the security review your code will not get.

> [!CAUTION]
> - **Do not implement your own password hashing, token format or crypto**
> - **Do not use long-lived stateless JWTs** as session tokens if logout must work
> - **Do not store session tokens in `localStorage`** for browser applications
> - **Do not build hard account lockout**, which is a denial-of-service feature
> - **Do not treat the reset flow as secondary** — it is a full authentication path
> - **Do not roll your own OAuth client** — use a certified library
> - **Do not skip the audit log**; you will need it during an incident

---

## 15. Advantages and Disadvantages

**Advantages of owning it**
- No per-user cost at scale
- Full control over the user experience and data model
- No vendor dependency in your most critical path
- Data stays where you want it

**Disadvantages**
- **You own every vulnerability**, in a domain where mistakes are severe
- MFA, passkeys, SSO and social login are each a project
- Security review and maintenance are permanent, not one-off
- Breach liability sits with you
- The features users now expect — visible active sessions, device management, passkeys — are a large surface

---

## 16. Performance Impact

| Aspect | Impact |
|---|---|
| **Password hashing** | 100–300 ms **by design** — never optimise this away |
| **Session lookup** | ~1 ms from Redis; do not query the primary database per request |
| **Stateless JWT verify** | Microseconds, and no revocation |
| **Login endpoint under attack** | Hashing is CPU-expensive — rate limiting protects your servers too |
| **Email and SMS** | Hundreds of milliseconds to seconds — always asynchronous |
| **Session store outage** | Everyone is logged out; treat it as tier-one infrastructure |

> [!CAUTION]
> **Deliberately slow hashing makes your login endpoint a CPU amplification target.** An attacker sending a thousand login attempts per second is asking you to perform a thousand argon2 computations per second, which will saturate the fleet regardless of whether the credentials are valid. Rate limiting at the edge is a capacity control, not only a security one.

---

## 17. Security Considerations

> [!CAUTION]
> **Assume your user table will leak one day, and design so that it is survivable.** Correct hashing turns a database dump from an immediate mass account takeover into an expensive offline cracking exercise for weak passwords only. This single decision determines whether a breach is an incident or a catastrophe.

- **argon2id** with tuned parameters; never a general-purpose fast hash
- **Constant-time comparison** for tokens and MFA codes
- **HttpOnly, Secure, SameSite cookies**; no session tokens in `localStorage`
- **Rotate refresh tokens** and treat reuse as theft
- **Invalidate all sessions** on password change, reset and email change
- **Confirm email changes at both addresses**
- **Audit log everything**, append-only, in a place a compromise cannot rewrite
- **Signing keys rotatable**, with a key id in the token so rotation is non-breaking
- **MFA enforced on every recovery path**, including support-driven ones
- **Alert on authentication anomalies** — failure spikes, impossible travel, one IP touching many accounts

---

## 18. Mental Model

> [!NOTE]
> **Login is the door; the session is the wristband.**
>
> Checking identity at the door is deliberately slow and thorough — that is fine, it happens once. After that the wristband is checked constantly, so it must be instant. The design question is what happens when a wristband is stolen: if the staff keep a list of valid bands (server-side sessions) they can void one immediately; if the band is simply printed with an expiry (a stateless JWT) it works until it expires, and nobody can stop it. Everything else — MFA, passkeys, rate limits — is about how hard it is to get through the door dishonestly.

---

## 19. Complete Request Flow

```text
─────────────── registration ───────────────
POST /register
    ↓
Password checked against a breached-password corpus → rejected if found
    ↓
Hashed with argon2id (~200 ms)
    ↓
User row created; verification email ENQUEUED (not sent inline)
    ↓
─────────────── login ───────────────
POST /login, rate limited at the edge by IP and by account
    ↓
User looked up. If absent, a DUMMY HASH is still computed
    → identical timing, identical error message
    ↓
Hash verified. MFA required → TOTP challenge issued
    ↓
6-digit code verified in constant time, attempts rate limited
    ↓
Session created in Redis; refresh token (family id, device, hashed) stored
    ↓
Access token in an HttpOnly, Secure, SameSite cookie (15 min)
Refresh token in a separate HttpOnly cookie (30 days)
    ↓
Audit log: login success, IP, device, method
    ↓
─────────────── every subsequent request ───────────────
Cookie sent automatically; CSRF token checked on state-changing requests
    ↓
Session validated against Redis (~1 ms)
    ↓
Handler receives an authenticated user id — AUTHORISATION is a separate concern
    ↓
─────────────── refresh ───────────────
Access token expires after 15 minutes
    ↓
Client calls /refresh with the refresh cookie
    ↓
Token hash matched; OLD TOKEN INVALIDATED; a new pair issued
    ↓
─────────────── token theft, detected ───────────────
An attacker replays a refresh token the legitimate client already rotated
    ↓
Reuse detected → the ENTIRE token family revoked
    ↓
Both parties forced to re-authenticate; the user is notified
    ↓
The stolen token bought the attacker minutes, not weeks
    ↓
─────────────── password reset ───────────────
POST /forgot-password → identical response whether the account exists or not
    ↓
High-entropy token generated; only its hash stored; 30-minute expiry, single use
    ↓
Email enqueued
    ↓
On successful reset: ALL sessions and refresh tokens invalidated
    ↓
Notification sent to the address on file
    ↓
─────────────── the breach that was survivable ───────────────
The users table is exfiltrated
    ↓
Attacker has emails and argon2id hashes
    ↓
Cracking is economically viable only for the weakest passwords
    ↓
Users with MFA or passkeys are unaffected regardless
    ↓
Forced reset for everyone; the audit log shows exactly what was accessed
```

---

## 20. Key Takeaway

> [!IMPORTANT]
> Delegate authentication if you can; if you build it, use argon2id, keep sessions server-side and revocable, rotate refresh tokens and treat reuse as theft, put session tokens in HttpOnly cookies, and design the reset flow with the same rigour as login.

---

## 21. Common Mistakes

- **A fast hash** (SHA-256, MD5) for passwords
- **Long-lived stateless JWTs as sessions**, making logout impossible
- **Session tokens in `localStorage`**, turning any XSS into account takeover
- **Different response times or messages** for existing versus unknown accounts
- **Password reset tokens** that are guessable, reusable or long-lived
- **Sessions left alive** after a password reset
- **Email change without confirming the old address**
- **Hard account lockout**, which is a denial-of-service feature
- **No refresh token rotation**, so a stolen token works indefinitely
- **MFA enforced on login but not on recovery**, making it bypassable
- **No rate limiting**, so credential stuffing and CPU exhaustion both work
- **No audit log**, so an incident cannot be reconstructed
- **Complexity rules instead of a breach list**, producing predictably weak passwords
- **Rolling your own crypto or OAuth client**
- **Sending email synchronously** inside the request

---

## 22. Open Source Technologies

- **Keycloak**, **Authentik**, **Ory Kratos**, **Zitadel** — self-hosted identity providers
- **Passport.js**, **Devise**, **Django auth**, **Spring Security**, **Lucia** — framework-level implementations
- **argon2**, **bcrypt** libraries — never implement the primitive yourself
- **SimpleWebAuthn**, **webauthn4j** — passkey support
- **oauthlib**, **openid-client**, **AppAuth** — certified OAuth and OIDC clients
- **Redis** — the session store
- **Have I Been Pwned API** — breached-password checking without sending the password
- **rate limiting**: nginx `limit_req`, Envoy, or Redis-backed token buckets
- **OWASP ASVS** and the **OWASP Authentication Cheat Sheet** — the checklist to review against

---

## 23. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 24. Workbook Exercise

- [ ] Time your login endpoint for an existing and a non-existent email. If they differ, you have an enumeration oracle.
- [ ] Check what hashing algorithm and cost your application uses.
- [ ] Ask: if a session token were stolen right now, how long until it stops working?
- [ ] Verify that a password reset invalidates existing sessions.
- [ ] Check whether MFA is enforced on the password reset path.
- [ ] Look at where your session token is stored in the browser.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
edge rate limit → auth service → users DB (argon2id) + session store (Redis)
                             → async email/SMS → append-only audit log
```

## 2. Request Flow

```text
Input       a credential once, then a session token on every request
    ↓
Processing  slow, constant-time verification at the door; ~1 ms revocable check thereafter
    ↓
Output      an authenticated identity — and the ability to revoke it instantly
```

## 3. Real-World Usage

The industry has converged on a clear pattern: delegate authentication where possible, use passkeys where users will accept them, keep sessions server-side and revocable, and treat password reset as a first-class authentication path. The systems that get breached are rarely broken at the cryptography — they are broken at recovery, at session lifetime, or at a support process nobody threat-modelled.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The system that establishes identity and then maintains it across requests |
| **Why does it exist?** | Because every other authorisation decision depends on it |
| **Where does it belong?** | In front of everything — ideally delegated to a specialist provider |
| **When should I use it?** | Always; build it yourself only for a specific, stated reason |

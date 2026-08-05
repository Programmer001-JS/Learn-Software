# JWT

> **In one line —** a signed, self-contained token carrying claims about a user; the server verifies a signature instead of looking anything up — which is its strength and its revocation problem.

| | |
|---|---|
| **Full name** | JSON Web Token |
| **Category** | Token Format |
| **Architectural Layer** | Authentication |
| **Pronounced** | "jot" |
| **Related notes** | [Authentication](Authentication.md) · [Session Store](../08%20-%20Databases%20and%20Data/Session%20Store.md) · [OAuth2](OAuth2.md) · [OpenID Connect](OpenID%20Connect.md) · [Microservices](../10%20-%20Distributed%20Systems/Microservices.md) |

---

## 1. Short Definition

*What is it?*

A JWT is a compact string with three base64url-encoded parts — header, payload, signature — carrying claims about an identity, signed so that tampering is detectable.

```text
eyJhbGciOiJIUzI1NiJ9 . eyJzdWIiOiI0MiIsImV4cCI6MTc0MH0 . 4f2a8b...
      header                    payload                    signature
```

---

## 2. Purpose

*What is its main purpose?*

To let any service verify who a caller is **without a shared session store** — by checking a signature rather than performing a lookup.

---

## 3. Problem

*What engineering problem does it solve?*

```text
SESSIONS ACROSS MANY SERVICES        JWT
every service must reach the         each service verifies the signature
shared session store                 with a public key
    ↓                                    ↓
a lookup per request                 no lookup, no shared dependency
the store is a single point          the token carries its own claims
of failure for everything
```

---

## 4. Architecture Position

```text
Client
    ↓  Authorization: Bearer eyJ...
API Gateway            verifies the signature once
    ↓
Service A · Service B · Service C
    ↓
each verifies independently — no shared store required
```

---

## 5. The critical misunderstanding

> [!CAUTION]
> **A JWT is signed, not encrypted.** The payload is base64 — anyone holding the token can read every claim in it, instantly, with no key. Paste one into jwt.io and see for yourself.
>
> **Never put anything sensitive in a JWT payload.** No personal data, no internal identifiers you would not publish, and obviously no secrets.

The signature proves two things and only two: the token was issued by someone holding the key, and it has not been modified since.

---

## 6. Structure

```json
// header
{ "alg": "RS256", "typ": "JWT" }

// payload — standard claims plus your own
{
  "sub": "42",                 // subject: who
  "iss": "https://auth.example.com",  // issuer
  "aud": "https://api.example.com",   // audience
  "exp": 1740000000,           // expiry  ← must be checked
  "iat": 1739996400,           // issued at
  "roles": ["customer"]
}
```

> [!IMPORTANT]
> A verifier must check **all** of: signature, `exp`, `iss` and `aud`. Checking only the signature accepts an expired token, or one issued by a different system entirely for a different audience.

---

## 7. The revocation problem

```text
User logs out / is banned / has their password changed
    ↓
The token is still valid until it expires
    ↓
There is nothing to delete — the server holds no state
```

Practical answers, in order of preference:

```text
1. SHORT-LIVED ACCESS TOKENS (5–15 min) + a refresh token
       the exposure window is bounded; refresh can be revoked
2. A BLOCKLIST of revoked token IDs (jti) in Redis
       ← reintroduces the lookup you removed
3. Rotate the signing key
       revokes every token at once — a blunt instrument
```

> [!IMPORTANT]
> If you end up with a Redis blocklist checked on every request, you have built a [session store](../08%20-%20Databases%20and%20Data/Session%20Store.md) with extra steps. That is not automatically wrong — but it is worth noticing, because sessions would have been simpler.

---

## 8. Access and refresh tokens

```text
ACCESS TOKEN     short (5–15 min), sent with every request, not stored server-side
REFRESH TOKEN    long (days), stored server-side, used only to obtain a new access token
    ↓
Logout or ban → delete the refresh token → access ends within minutes
```

> [!TIP]
> This is the standard pattern and it resolves most of the revocation objection. **Refresh tokens must be stored and revocable** — a stateless refresh token gives you no revocation at all.

---

## 9. Real World Example

- **[OpenID Connect](OpenID%20Connect.md)** ID tokens are JWTs — "Sign in with Google" hands you one.
- **API gateways** verify a JWT once at the edge and forward identity to internal services.
- **Service-to-service calls** in microservice systems carry short-lived JWTs.
- **Mobile applications**, where cookies are awkward, commonly use bearer tokens.

---

## 10. Where to store it in a browser

```text
localStorage       readable by ANY JavaScript on the page
                   → one XSS = full account takeover
                   ✗ avoid for auth tokens

httpOnly cookie    not readable by JavaScript
                   → needs SameSite / CSRF protection
                   ✓ safer default

In memory only     cleared on refresh; strong, with UX cost
```

> [!CAUTION]
> "Use JWTs in localStorage to avoid CSRF" trades a well-understood, well-defended risk (CSRF, solved by `SameSite`) for a far more damaging one (XSS token theft). For browser applications, an `httpOnly` cookie is the better choice — and at that point, a session may be simpler still.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use JWTs for **stateless verification across service or organisational boundaries**: microservices, mobile clients, third-party API access, and anywhere an identity provider issues tokens your systems must verify independently.

> [!CAUTION]
> Prefer [sessions](../08%20-%20Databases%20and%20Data/Session%20Store.md) for a single browser-facing application. You get instant revocation, a tiny cookie, and no need to reason about `exp` windows. The stateless property only pays off when there is genuinely no shared store to reach.

---

## 12. Advantages and Disadvantages

**Advantages**
- No lookup per request — verification is local
- Works across services, languages and organisations
- Carries claims, so downstream services need no user fetch
- Standardised, with mature libraries everywhere

**Disadvantages**
- **Revocation is genuinely hard**
- Larger than a session ID, sent on every request
- Payload is readable by anyone holding it
- Easy to misconfigure in ways that are catastrophic
- Claims go stale — a role change is not reflected until expiry

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Verification** | HS256 is trivial; RS256 costs slightly more but avoids sharing a secret |
| **Network** | 200–800 bytes on every request, versus ~40 for a session ID |
| **No lookup** | The main benefit — no round trip to a store |
| **Blocklist** | Adds back the lookup you were avoiding |

---

## 14. Security Considerations

> [!CAUTION]
> **The `alg: none` attack** — early libraries accepted a token declaring no algorithm and skipped verification entirely. Related: an attacker changes `RS256` to `HS256` and signs with the public key as the HMAC secret. **Always pin the expected algorithm explicitly**; never trust the header to tell you how to verify.

- **Never trust the header's `alg`** — configure the accepted algorithm on the verifier
- **Verify `exp`, `iss` and `aud`**, not only the signature
- **Use a strong secret** — HS256 with a short secret is brute-forceable offline
- **Prefer RS256/ES256** across trust boundaries, so verifiers hold only a public key
- **Nothing sensitive in the payload**
- **Short expiry plus revocable refresh tokens**
- **Rotate signing keys**, and publish them via JWKS so rotation does not require a deployment

---

## 15. Mental Model

> [!NOTE]
> **A JWT is a festival wristband with a hologram.**
>
> The staff check the hologram, not a guest list — fast, and it works at every gate without a central computer. But once it is on, you cannot un-issue it: the only real controls are printing an expiry date on it and making it short.

---

## 16. Mini Architecture Diagram

```text
Auth server
    ↓ signs with a private key
JWT → client
    ↓ Authorization: Bearer
API Gateway ── verifies with the public key (JWKS)
    ↓
Service A   Service B   Service C
    ↓
each verifies independently: signature · exp · iss · aud
```

---

## 17. Complete Request Flow

```text
─────────── login ───────────
Credentials verified
    ↓
Access token issued  (exp: 15 min)
Refresh token issued (exp: 7 days, STORED server-side)
    ↓
─────────── every request ───────────
Authorization: Bearer <access token>
    ↓
Verify signature with the pinned algorithm and the public key
    ↓
Check exp, iss, aud → 401 on any failure
    ↓
Claims used for authorisation — plus an object-level ownership check
    ↓
─────────── access token expires ───────────
Client presents the refresh token
    ↓
Server checks it is still valid AND not revoked  ← the stateful part
    ↓
New access token issued
    ↓
─────────── logout ───────────
Refresh token deleted → access ends within 15 minutes
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> A JWT is readable by anyone and revocable by no one — keep access tokens short, keep refresh tokens stateful, and pin the verification algorithm.

---

## 19. Common Mistakes

- **Believing the payload is encrypted**
- **Putting personal or sensitive data in claims**
- **Storing tokens in localStorage** where XSS can read them
- **Not verifying `exp`, `iss` and `aud`**
- **Trusting the `alg` header**
- **Long-lived access tokens** with no revocation path
- **Stateless refresh tokens**, which cannot be revoked either
- **Choosing JWTs for a single web application** where sessions are simpler and safer

---

## 20. Open Source Technologies

- **PyJWT**, **jsonwebtoken** (Node), **jjwt** (Java), **System.IdentityModel.Tokens.Jwt**
- **Keycloak**, **Ory Hydra**, **Authentik**, **SuperTokens** — issuers
- **JWKS** — public key distribution for rotation
- **PASETO** — a JWT alternative designed to remove the algorithm-confusion class of bugs
- **jwt.io** — decode a token and see exactly how readable it is

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Decode one of your own tokens at jwt.io and check whether anything in it should not be public.
- [ ] Verify your library pins the expected algorithm rather than reading it from the header.
- [ ] Work out how long a banned user would retain access with your current token lifetimes.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Auth server → JWT → client → gateway/services (independent verification)
```

## 2. Request Flow

```text
Input       a bearer token presented with a request
    ↓
Processing  signature verified with a pinned algorithm; exp, iss and aud checked
    ↓
Output      an authenticated identity with claims — no store lookup
```

## 3. Real-World Usage

**"Sign in with Google"** issues an OpenID Connect ID token, which is a JWT. Your application verifies it with Google's published keys and never handles a password — a good illustration of what JWTs are genuinely well suited to.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A signed, self-contained token carrying identity claims |
| **Why does it exist?** | So services can verify identity without a shared session store |
| **Where does it belong?** | Across service and organisational boundaries |
| **When should I use it?** | Multi-service and mobile scenarios — sessions for a single web app |

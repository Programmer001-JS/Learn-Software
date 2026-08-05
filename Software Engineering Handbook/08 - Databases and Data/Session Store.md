# Session Store

> **In one line —** where the server keeps "who is logged in", so that any server can answer any request and logging someone out actually works.

| | |
|---|---|
| **Category** | Architectural Component |
| **Architectural Layer** | Data / Auth |
| **Related notes** | [Redis](Redis.md) · [Authentication](../09%20-%20Security/Authentication.md) · [JWT](../09%20-%20Security/JWT.md) · [HTTP HTTPS](../04%20-%20Networking%20and%20Internet/HTTP%20HTTPS.md) |

---

## 1. Short Definition

*What is it?*

A session store holds server-side state about a logged-in user. The client carries only an opaque **session ID** in a cookie; everything meaningful lives on the server, keyed by that ID.

---

## 2. Purpose

*What is its main purpose?*

To make [HTTP's statelessness](../04%20-%20Networking%20and%20Internet/HTTP%20HTTPS.md) usable for applications that need to remember who someone is between requests.

---

## 3. Problem

*What engineering problem does it solve?*

HTTP forgets everything between requests. The client must therefore prove its identity every time. Sending credentials with each request is unacceptable, so instead the server issues a token that maps to state it holds.

```text
Login
    ↓
Server creates a session, stores { user_id, roles, expiry }
    ↓
Returns:  Set-Cookie: session=8f3a...   ← an opaque random ID, nothing more
    ↓
Every later request carries that cookie
    ↓
Server looks the ID up and knows who it is
```

---

## 4. Architecture Position

```text
Browser  (cookie: session=8f3a...)
    ↓
Load balancer
    ↓
App server 1     App server 2     App server 3
    └────────────────┼────────────────┘
                     ↓
              SESSION STORE  (Redis)
                     ↓
                 Database
```

> [!IMPORTANT]
> The session store must be **shared**, not in the memory of one application server. In-memory sessions mean a user is tied to the server that created them — which breaks load balancing, rolling deploys and any restart.

---

## 5. Where sessions can live

| Store | Suitability |
|---|---|
| **Application memory** | Single server only. Breaks on restart and on scaling. |
| **[Redis](Redis.md)** | The standard answer — fast, shared, TTL built in |
| **Database** | Works, durable, but a query on every request |
| **Encrypted cookie** | No server storage; but then it is a token, not a session |
| **Sticky sessions at the LB** | A workaround, not a solution — a lost server logs its users out |

> [!TIP]
> Redis is the default for a reason: sessions are ephemeral, read on every request, and benefit directly from a native TTL that expires them without a cleanup job.

---

## 6. Sessions vs JWT — the real trade

```text
SESSION (server state)                 JWT (client state)
opaque ID in a cookie                  signed claims in the token
lookup on every request                verify signature, no lookup
    ↓                                      ↓
REVOCATION IS INSTANT                  revocation needs a blocklist —
delete the key, they are out           which reintroduces the lookup
scales to any number of servers        stateless across services
store is a dependency                  token size sent on every request
```

> [!IMPORTANT]
> The decisive question is **revocation**. If "log this user out now" or "ban this account immediately" must work, sessions do it trivially and JWTs do not. Many teams choose JWTs for statelessness, then add a Redis blocklist for revocation — at which point they have a session store with extra steps.

See [JWT](../09%20-%20Security/JWT.md).

---

## 7. What belongs in a session

```text
✓ user_id                    the essential item
✓ roles or permissions       if cheap to recompute, prefer fetching instead
✓ CSRF token
✓ last activity timestamp
✓ small UI state (locale, theme)

✗ the full user object       stale the moment the profile changes
✗ large data or shopping baskets   memory cost per active user
✗ anything sensitive you would not want in a memory dump
```

---

## 8. Real World Example

- **Django, Laravel, Rails, Express** all ship session middleware with pluggable stores; switching to Redis is a configuration line.
- **PHP's default file-based sessions** are the classic reason a PHP application cannot be scaled to two servers without a change.
- **Sticky sessions** appear in most load balancers precisely because so many applications kept sessions in memory.

---

## 9. Communication and Dependencies

- **A cookie** carrying the session ID
- **[Redis](Redis.md)** or another shared store
- **Session middleware** in your framework
- **A TTL** — expiry is not optional

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Use server-side sessions for browser-based applications, especially where instant revocation, small tokens and simple invalidation matter. For a monolith or a small number of services, this is the simpler and safer default.

> [!CAUTION]
> Sessions fit poorly when many independent services must verify identity without sharing a store, or for mobile and machine-to-machine clients where cookies are awkward. Those are the cases JWTs were designed for.

---

## 11. Advantages and Disadvantages

**Advantages**
- Instant revocation — delete the key
- Tiny cookie; no data on the client
- Session contents can be changed server-side at any moment
- Simple to reason about and to audit

**Disadvantages**
- A store lookup on every request
- The store is a hard dependency — if Redis is down, everyone is logged out
- Memory grows with concurrent users
- Cross-domain and mobile use is more awkward than a bearer token

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Per request** | One Redis GET — well under a millisecond |
| **Memory** | A few hundred bytes to a few KB per active session |
| **Failure** | Store unavailable = every user logged out at once |
| **TTL** | Handles cleanup automatically; without it, memory grows forever |

---

## 13. Security Considerations

> [!CAUTION]
> **Session hijacking** — a stolen session ID is a stolen account. Everything below exists to make that harder, and each one has been the root cause of real breaches.

**Cookie flags — all three are required:**

```text
HttpOnly    JavaScript cannot read it  → XSS cannot steal the session
Secure      HTTPS only                 → not sent over plaintext
SameSite    Lax or Strict              → CSRF protection
```

- **Regenerate the session ID on login** — otherwise **session fixation**: an attacker sets a known ID before login and inherits the authenticated session
- **Use a cryptographically random ID** with enough entropy (128 bits or more)
- **Set both an absolute and an idle timeout** — a session valid forever is a permanent credential
- **Invalidate all sessions on password change** — otherwise a compromised account stays compromised after the reset
- **Never store sensitive data in the session** in plaintext — a Redis compromise is then a data breach as well as an authentication one

---

## 14. Mental Model

> [!NOTE]
> **A session is a cloakroom ticket.**
>
> The ticket is a meaningless number. The coat stays with the cloakroom. Lose the ticket and someone else can collect your coat — which is why the ticket must be hard to guess and hard to steal. And the cloakroom can refuse a ticket at any moment, which is exactly what revocation means.

---

## 15. Mini Architecture Diagram

```text
Browser
  cookie: session=8f3a  (HttpOnly, Secure, SameSite)
    ↓
Any app server
    ↓
Redis:  GET session:8f3a  →  { user_id: 42, roles: [...] }
    ↓
Request proceeds as user 42
```

---

## 16. Complete Request Flow

```text
POST /login with credentials
    ↓
Password verified (hashed comparison)
    ↓
SESSION ID REGENERATED — old one discarded (fixation defence)
    ↓
Redis: SET session:<new id> {...} EX 1800
    ↓
Set-Cookie: session=<id>; HttpOnly; Secure; SameSite=Lax
    ↓
─────────── every later request ───────────
Cookie sent automatically
    ↓
Middleware: GET session:<id>
    ↓
Missing or expired → 401, redirect to login
    ↓
Found → user attached to the request, TTL refreshed
    ↓
─────────── logout ───────────
DEL session:<id>  → effective immediately, everywhere
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> A session store keeps identity on the server so the cookie is a meaningless ticket — which is what makes instant revocation possible and why it must be shared between all servers.

---

## 18. Common Mistakes

- **Sessions in application memory**, breaking scaling and restarts
- **Missing `HttpOnly`, `Secure` or `SameSite`** flags
- **Not regenerating the ID on login** — session fixation
- **No expiry**, creating permanent credentials
- **Storing large or sensitive objects** in the session
- **Not invalidating sessions on password change**
- **Sticky sessions** used to paper over in-memory storage

---

## 19. Open Source Technologies

- **Redis**, **Valkey** — the usual store
- **connect-redis** (Express), **django-redis**, **Laravel Redis driver**
- **Spring Session** — pluggable session storage for the JVM
- **OWASP Session Management Cheat Sheet** — the reference for the security rules above

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Inspect your session cookie in DevTools and confirm all three security flags are set.
- [ ] Check whether your application regenerates the session ID on login.
- [ ] Work out what happens to logged-in users if your session store restarts.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Browser (cookie) → any app server → shared session store (Redis) → database
```

## 2. Request Flow

```text
Input       a request carrying an opaque session ID
    ↓
Processing  the ID is looked up in the shared store; identity attached
    ↓
Output      an authenticated request, revocable at any instant
```

## 3. Real-World Usage

**PHP's default file-based sessions** are the textbook example of why the store must be shared: an application works perfectly on one server and logs users out randomly the moment a second is added behind a load balancer.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Server-side storage of logged-in user state, keyed by an opaque ID |
| **Why does it exist?** | Because HTTP is stateless but applications need identity |
| **Where does it belong?** | In a shared store reachable by every application server |
| **When should I use it?** | Browser applications, and anywhere instant revocation matters |

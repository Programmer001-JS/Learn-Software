# Middleware

> **In one line —** code that runs on every request before and after your handler, so cross-cutting concerns live in one place instead of being copy-pasted into every endpoint.

| | |
|---|---|
| **Category** | Architectural Pattern |
| **Architectural Layer** | Application, request pipeline |
| **Related notes** | [Routing](Routing.md) · [Validation](Validation.md) · [Controllers](Controllers.md) · [Backend Fundamentals](../06%20-%20Backend%20Architecture/Backend%20Fundamentals.md) · [Authentication](../09%20-%20Security/Authentication.md) |

---

## 1. Short Definition

*What is it?*

Middleware is a function that sits in a chain between the incoming request and your handler. Each one can inspect or modify the request, pass it along, and then inspect or modify the response on the way back.

---

## 2. Purpose

*What is its main purpose?*

To handle **cross-cutting concerns** — things every endpoint needs but none of them is about: logging, authentication, CORS, compression, rate limiting, error handling.

---

## 3. Problem

*What engineering problem does it solve?*

```text
WITHOUT MIDDLEWARE                    WITH MIDDLEWARE

def get_user():                       app.use(log)
    log(request)                      app.use(authenticate)
    check_auth()                      app.use(rate_limit)
    check_rate_limit()
    ...actual logic...                def get_user():
                                          ...actual logic...    ← only the logic
def get_order():
    log(request)                      ← the same four lines
    check_auth()                        repeated in every handler,
    check_rate_limit()                  and forgotten in one of them
    ...actual logic...
```

> [!IMPORTANT]
> The real risk without middleware is not duplication — it is **omission**. Someone adds a new endpoint and forgets the auth check. Middleware applies by default, which is exactly what a security control should do.

---

## 4. Architecture Position

```text
Request
    ↓
┌──── MIDDLEWARE PIPELINE ────┐
│  request ID                  │
│  logging                     │
│  CORS                        │
│  rate limiting               │
│  authentication              │
│  authorisation               │
└──────────────┬───────────────┘
               ↓
           Router → Controller
               ↓
┌──── the same pipeline, in reverse ────┐
│  response headers, compression, timing │
└──────────────┬─────────────────────────┘
               ↓
           Response
```

---

## 5. The onion model

Middleware wraps the handler. Code before `next()` runs on the way in; code after it runs on the way out.

```javascript
async function timing(req, res, next) {
  const start = Date.now();      // ← on the way IN
  await next();                  // ← everything inside
  const ms = Date.now() - start; // ← on the way OUT
  logger.info({ path: req.path, ms });
}
```

```text
  ┌───────── logging ─────────┐
  │  ┌────── auth ─────────┐  │
  │  │  ┌── handler ──┐    │  │
  │  │  └─────────────┘    │  │
  │  └─────────────────────┘  │
  └───────────────────────────┘
```

---

## 6. Order is everything

> [!CAUTION]
> Middleware order is not a style question — it changes behaviour and security.

```text
CORRECT                          BROKEN
error handler (outermost)        authorisation
request ID                       authentication      ← authorises before knowing who
logging                                                the user is: always fails or
CORS                                                   always passes
rate limiting                    rate limiting
authentication                   ...after auth, so unauthenticated
authorisation                       requests are never limited
handler
```

Rules that hold in every framework:

- **Authentication before authorisation** — you cannot check permissions for an unknown user
- **Rate limiting before authentication** — otherwise brute-force attempts are never throttled
- **Error handling outermost** — so it catches everything inside
- **Request ID first** — so every log line downstream can be correlated

---

## 7. Typical middleware

| Middleware | Job |
|---|---|
| **Request ID** | Attach a correlation ID for logs and error responses |
| **Logging** | Method, path, status, duration |
| **CORS** | Which origins may call this API from a browser |
| **Body parsing** | JSON/form → an object, with a size limit |
| **Rate limiting** | Throttle abusive callers |
| **Authentication** | Identify the caller |
| **Authorisation** | Decide whether they may proceed |
| **Compression** | gzip/brotli responses |
| **Error handling** | Map exceptions to status codes |

---

## 8. Real World Example

- **Express** is built entirely on this model — routes are middleware too.
- **ASP.NET Core** calls it the middleware pipeline; **NestJS** splits it into middleware, guards, interceptors and pipes.
- **Django** middleware is where CSRF, sessions and security headers live.
- **Service meshes** (Envoy, Linkerd) apply the same idea at the network layer, outside your process.

---

## 9. Communication and Dependencies

- **The framework** — each defines its own signature and ordering rules
- **Everything downstream** — middleware often attaches data (`req.user`, `HttpContext.User`) that handlers rely on
- **Observability** — logs, metrics and traces are usually produced here

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Use middleware for anything that applies to **most** requests and is not about a specific endpoint's business rules.

> [!CAUTION]
> Do not put business logic in middleware. It becomes invisible — a rule enforced somewhere no one reading the handler will see, and impossible to test in isolation. If it applies to one endpoint, it belongs in that endpoint.

---

## 11. Advantages and Disadvantages

**Advantages**
- Cross-cutting concerns defined once, applied by default
- Impossible to forget on a new endpoint
- Composable and reorderable
- Testable in isolation

**Disadvantages**
- Control flow becomes implicit — hard to see from the handler
- Order-dependent bugs are subtle and easy to introduce
- Every middleware runs on every request, so cost accumulates
- Deep chains make debugging and stack traces harder

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Latency** | Each layer adds a small cost; ten layers on every request is measurable |
| **Ordering** | Cheap rejections (rate limit, auth) should come **before** expensive work |
| **Scope** | Apply heavy middleware only to routes that need it |

> [!TIP]
> Put the cheapest rejections first. Parsing a 10 MB body before checking a rate limit means the attacker got you to do the expensive part anyway.

---

## 13. Security Considerations

> [!CAUTION]
> Middleware is where most security controls live, which makes **misordering or scoping it wrongly** a direct vulnerability. An auth middleware registered after a route in Express simply never runs for that route — with no error and no warning.

- **Apply security middleware globally**, then opt specific routes out deliberately — never the reverse
- **Set body size limits** before parsing
- **Never log request bodies or headers wholesale** — passwords and tokens end up in logs
- **Error middleware must not leak stack traces** to clients
- **Validate `X-Forwarded-For`** handling, or rate limiting can be bypassed by forged headers

---

## 14. Mental Model

> [!NOTE]
> **Middleware is airport security.**
>
> Every passenger passes through the same checks in a fixed order — ticket, then ID, then screening — before reaching the gate. Nobody at the gate re-checks the passport, because the process guarantees it already happened. Change the order and the whole thing stops working.

---

## 15. Mini Architecture Diagram

```text
Request
  ↓ request ID
  ↓ logging
  ↓ CORS
  ↓ body parse (size-limited)
  ↓ rate limit          → 429
  ↓ authentication      → 401
  ↓ authorisation       → 403
  ↓
HANDLER
  ↑
  ↑ response shaping
  ↑ compression
  ↑ timing log
  ↑ error handler       → 500
Response
```

---

## 16. Complete Request Flow

```text
POST /orders arrives
    ↓
Request ID generated → attached to the request and every log line
    ↓
Rate limit checked → 429 if exceeded          ← cheap rejection, done early
    ↓
Body parsed, capped at 100 KB
    ↓
Authentication: token verified → req.user set, or 401
    ↓
Authorisation: may this user create orders? → or 403
    ↓
HANDLER: business logic
    ↓
Response passes back out: headers added, compressed, duration logged
    ↓
An exception anywhere → error middleware → clean 500 with the request ID
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> Middleware applies cross-cutting concerns by default so they cannot be forgotten — and its order is a functional and security decision, not a formatting one.

---

## 18. Common Mistakes

- **Wrong order** — authorisation before authentication, or rate limiting after auth
- **Middleware registered after routes**, so it silently never runs
- **Business logic hidden in middleware**
- **Logging entire request bodies**, capturing credentials
- **Expensive middleware before cheap rejections**
- **Opting routes in to security** rather than opting them out
- **Error middleware exposing stack traces**

---

## 19. Open Source Technologies

- **helmet**, **cors**, **express-rate-limit** — Express middleware
- **Starlette middleware** — FastAPI
- **Django middleware** — sessions, CSRF, security headers
- **ASP.NET Core middleware**, **NestJS guards and interceptors**
- **Envoy**, **Linkerd** — the same pattern at the network layer

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Write out your middleware chain in order and check the four rules in section 6.
- [ ] Verify that adding a brand-new route automatically gets authentication applied.
- [ ] Add a request ID middleware and confirm the ID appears in both logs and error responses.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Request → middleware chain → router → handler → middleware (reverse) → response
```

## 2. Request Flow

```text
Input       an HTTP request
    ↓
Processing  each middleware inspects, modifies, and passes it along
    ↓
Output      a response shaped on the way back out
```

## 3. Real-World Usage

**Express's** entire design is this pattern, and its most common production bug is ordering: a security middleware registered after a route never protects it. The framework gives no warning, which is why the ordering rules are worth memorising.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Chained functions running before and after every request handler |
| **Why does it exist?** | So cross-cutting concerns apply by default and cannot be forgotten |
| **Where does it belong?** | Between the server and your handlers |
| **When should I use it?** | For anything affecting most requests — never for endpoint-specific logic |

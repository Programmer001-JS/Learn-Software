# HTTP / HTTPS

> **In one line —** the request/response language of the web; HTTPS is the same language spoken inside an encrypted tunnel.

| | |
|---|---|
| **Full name** | HyperText Transfer Protocol (Secure) |
| **Category** | Application Protocol |
| **Architectural Layer** | Application |
| **Related notes** | [TLS SSL](TLS%20SSL.md) · [TCP IP](TCP%20IP.md) · [API Design](API%20Design.md) · [WebSockets](WebSockets.md) · [Authentication](../09%20-%20Security/Authentication.md) |

---

## 1. Short Definition

*What is it?*

HTTP is a text-based protocol in which a client sends a **request** and a server returns a **response**. It is **stateless** — the server remembers nothing between requests unless the application arranges it. HTTPS is HTTP carried inside a [TLS](TLS%20SSL.md) encrypted connection.

---

## 2. Purpose

*What is its main purpose?*

To transfer representations of resources between clients and servers in a way that is simple, cacheable and independent of what the data actually is.

---

## 3. Problem

*What engineering problem does it solve?*

Clients and servers written by different people, in different languages, on different platforms, need a shared convention for "give me this" and "here it is, or here is why not". HTTP's success comes from being simple enough that anything can implement it.

---

## 4. Architecture Position

```text
Your application / API
    ↓
HTTP            ← you are here: methods, headers, status codes
    ↓
TLS             (this is what makes it HTTPS)
    ↓
TCP             reliable byte stream
    ↓
IP → network
```

---

## 5. Anatomy of a request and response

```text
REQUEST                              RESPONSE

POST /api/orders HTTP/1.1            HTTP/1.1 201 Created
Host: example.com                    Content-Type: application/json
Authorization: Bearer eyJ...         Location: /api/orders/42
Content-Type: application/json       Cache-Control: no-store

{"item": "book", "qty": 2}           {"id": 42, "status": "pending"}
```

Three parts in each: a start line, headers, and an optional body.

---

## 6. Methods and what they promise

| Method | Purpose | Safe? | Idempotent? |
|---|---|---|---|
| **GET** | Retrieve | Yes | Yes |
| **POST** | Create / act | No | **No** |
| **PUT** | Replace entirely | No | Yes |
| **PATCH** | Modify partially | No | No |
| **DELETE** | Remove | No | Yes |

> [!IMPORTANT]
> **Idempotent** means calling it twice has the same effect as calling it once. This is not academic: networks retry, users double-click, and mobile clients resend. A non-idempotent `POST /payments` that gets retried charges the customer twice — which is why idempotency keys exist.

---

## 7. Status codes

```text
2xx  SUCCESS          200 OK · 201 Created · 204 No Content
3xx  REDIRECT         301 Moved Permanently · 304 Not Modified
4xx  CLIENT ERROR     400 Bad Request · 401 Unauthorized · 403 Forbidden
                      404 Not Found · 409 Conflict · 422 Unprocessable · 429 Too Many Requests
5xx  SERVER ERROR     500 Internal · 502 Bad Gateway · 503 Unavailable · 504 Timeout
```

> [!TIP]
> **401 means "I do not know who you are"; 403 means "I know who you are and you may not."** Confusing the two is one of the most common API design mistakes, and it makes client-side error handling impossible to get right.

---

## 8. Statelessness

Every request must carry everything the server needs. The server holds no memory of previous requests.

```text
Request 1:  GET /profile   +  Authorization: Bearer ...
Request 2:  GET /orders    +  Authorization: Bearer ...   ← identity re-sent every time
```

> [!IMPORTANT]
> Statelessness is what makes horizontal scaling trivial: any server can handle any request. It is also why [sessions](../08%20-%20Databases%20and%20Data/Session%20Store.md) and [JWTs](../09%20-%20Security/JWT.md) exist — they are how applications add state back on top of a stateless protocol.

---

## 9. HTTP versions

| Version | Key change | Impact |
|---|---|---|
| **HTTP/1.1** | Keep-alive, one request at a time per connection | Head-of-line blocking; browsers opened 6 connections per host |
| **HTTP/2** | Binary framing, multiplexing, header compression | Many parallel streams on one connection |
| **HTTP/3** | Runs over QUIC/UDP instead of TCP | Removes TCP head-of-line blocking; faster connection setup |

---

## 10. Caching — the most underused feature

```text
Client requests a resource
    ↓
Response includes:  Cache-Control: max-age=3600
                    ETag: "abc123"
    ↓
Within an hour: served from the local cache, no request at all
    ↓
After that: GET with If-None-Match: "abc123"
    ↓
Unchanged → 304 Not Modified, empty body   ← full round trip, near-zero bytes
```

---

## 11. Real World Example

- **Every REST API** in existence is HTTP conventions applied to a domain.
- **CDNs** are HTTP caches placed near users; the `Cache-Control` header is what makes them work.
- **Browsers block insecure requests** from HTTPS pages, which is why mixed content breaks sites.
- **`429 Too Many Requests` with `Retry-After`** is how well-behaved APIs communicate rate limits.

---

## 12. HTTP vs HTTPS

```text
HTTP                              HTTPS
plaintext on the wire             encrypted by TLS
anyone in the path can read       only the endpoints can read
and MODIFY the content            tampering is detectable
no server identity check          certificate proves who the server is
```

> [!CAUTION]
> Plain HTTP is not merely "less private" — it is **modifiable in transit**. ISPs have injected advertisements into plain HTTP pages, and attackers on shared Wi-Fi can rewrite responses. There is no legitimate reason to serve public HTTP today; certificates are free via Let's Encrypt.

---

## 13. When To Use

> [!TIP]
> Use HTTP/HTTPS for anything request/response between systems: APIs, web pages, webhooks, service-to-service calls. It is the default for a reason — universal support, caching, proxies, and tooling everywhere.

---

## 14. When NOT To Use

> [!CAUTION]
> - **Real-time bidirectional communication** — use [WebSockets](WebSockets.md); polling an HTTP endpoint every second wastes enormous resources.
> - **High-throughput internal service-to-service calls** — [gRPC](gRPC.md) is more efficient over the same connection.
> - **Streaming large media** — specialised protocols handle adaptive bitrate far better.

---

## 15. Advantages and Disadvantages

**Advantages**
- Universally supported by every language and platform
- Stateless, therefore trivially scalable horizontally
- Rich caching semantics built in
- Human-readable and easy to debug
- Works through firewalls and proxies everywhere

**Disadvantages**
- Verbose — headers repeat on every request (fixed in HTTP/2)
- Request/response only; the server cannot initiate
- Statelessness pushes session management into the application
- Text parsing overhead compared with binary protocols

---

## 16. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Dominated by round trips: DNS + TCP + TLS + request |
| **Memory** | Minimal per request |
| **CPU** | TLS costs a little; header parsing is cheap |
| **Network** | Headers can exceed the body for small API responses |

> [!TIP]
> The largest HTTP performance wins are almost always **fewer requests**, **connection reuse** and **correct cache headers** — not faster server code.

---

## 17. Security Considerations

> [!CAUTION]
> HTTP itself provides no security. Everything below depends on HTTPS being enforced, not merely available.

- **Always redirect HTTP → HTTPS** and set **HSTS** so browsers refuse plaintext entirely
- **Secrets in URLs are logged** — by servers, proxies and browser history. Put tokens in headers, never in query strings.
- **CORS** controls which origins may read responses in a browser; a permissive `Access-Control-Allow-Origin: *` on an authenticated API is a real vulnerability
- **Security headers**: `Content-Security-Policy`, `X-Content-Type-Options`, `Strict-Transport-Security`
- **Cookies** need `Secure`, `HttpOnly` and `SameSite`
- **Never return stack traces** in 5xx responses; they leak paths and versions

---

## 18. Mental Model

> [!NOTE]
> **HTTP is ordering by letter from a catalogue.**
>
> You write exactly what you want, include your membership number every single time (the shop keeps no record of you between letters), and receive either the goods or a numbered rejection slip explaining why not. **HTTPS is the same letter in a sealed, tamper-evident envelope** — and the shop's seal proves the reply really came from them.

---

## 19. Mini Architecture Diagram

```text
Browser / client
    ↓
DNS → IP
    ↓
TCP connection
    ↓
TLS handshake        ← the "S" in HTTPS
    ↓
HTTP request  ──────────────►  Server
HTTP response ◄──────────────
    ↓
Cached per Cache-Control
```

---

## 20. Complete Request Flow

```text
GET /api/orders/42
    ↓
Cached locally and still fresh?  → return, no network at all
    ↓ no
DNS → TCP handshake → TLS handshake
    ↓
Request sent with Authorization header
    ↓
Reverse proxy / CDN — can it serve this?  → 200 from cache
    ↓ no
Application server: authenticate → authorise → query database
    ↓
200 OK  +  Cache-Control  +  ETag  +  JSON body
    ↓
Client stores it; the next request sends If-None-Match → likely 304
```

---

## 21. Key Takeaway

> [!IMPORTANT]
> HTTP is a stateless request/response protocol whose statelessness makes scaling easy — and HTTPS is not optional, because plain HTTP can be read *and rewritten* by anyone in the path.

---

## 22. Common Mistakes

- **Confusing 401 and 403**
- **Using POST for everything**, discarding idempotency and caching
- **Returning 200 with an error inside the body** — clients and proxies cannot see it
- **Putting tokens in query strings**, where they end up in logs
- **Ignoring cache headers** entirely and re-fetching unchanged data forever
- **Permissive CORS** on authenticated endpoints
- **Serving plain HTTP** anywhere in production

---

## 23. Open Source Technologies

- **Nginx**, **Caddy**, **HAProxy** — HTTP servers and reverse proxies
- **curl**, **httpie**, **Postman** — clients for testing
- **OpenAPI / Swagger** — describing HTTP APIs formally
- **Let's Encrypt** — free certificates, the reason HTTPS became universal

---

## 24. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 25. Workbook Exercise

- [ ] Run `curl -v https://example.com` and identify every phase in the output.
- [ ] Review your own API and check whether each endpoint returns a semantically correct status code.
- [ ] Add correct `Cache-Control` headers to one endpoint and measure the difference.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Client
    ↓
HTTP
    ↓
TLS
    ↓
TCP → IP → network
    ↓
Server
```

## 2. Request Flow

```text
Input       a method, path, headers and optional body
    ↓
Processing  routed, authenticated, authorised, handled
    ↓
Output      a status code, headers and an optional body
```

## 3. Real-World Usage

**Cloudflare and every other CDN** work entirely through HTTP caching semantics. Setting `Cache-Control` correctly on a static asset means it is served from a machine in the user's city instead of from your origin server — the single highest-leverage web performance change available.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A stateless request/response protocol, encrypted by TLS when it is HTTPS |
| **Why does it exist?** | To give heterogeneous systems one simple, cacheable convention |
| **Where does it belong?** | The application layer, above TLS and TCP |
| **When should I use it?** | Nearly all client-server communication — but not real-time or high-throughput internal calls |

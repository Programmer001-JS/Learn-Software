# Web Servers

> **In one line —** the program that accepts connections and decides what happens next; usually two of them stacked, and knowing which does what removes most deployment confusion.

| | |
|---|---|
| **Category** | Overview note *(hub for this sub-section)* |
| **Architectural Layer** | Server, in front of the application |
| **Sub-topics** | [Nginx](Nginx.md) · [Gunicorn](Gunicorn.md) · [Uvicorn](Uvicorn.md) |
| **Related notes** | [Backend Fundamentals](Backend%20Fundamentals.md) · [Load Balancing](../14%20-%20Scalability%20and%20Reliability/Load%20Balancing.md) · [TLS SSL](../04%20-%20Networking%20and%20Internet/TLS%20SSL.md) |

---

## 1. Short Definition

A web server accepts incoming HTTP connections and either serves a response itself (static files) or passes the request to an application. In practice there are **two distinct roles**, and they are frequently confused.

---

## 2. The two roles

```text
REVERSE PROXY / EDGE SERVER          APPLICATION SERVER
Nginx, Caddy, HAProxy                 Uvicorn, Gunicorn, Kestrel, Tomcat
    ↓                                     ↓
TLS termination                       runs YOUR code
static files                          one per language ecosystem
compression, caching                  manages workers
load balancing                        speaks the framework's interface
rate limiting                         (WSGI, ASGI, servlet, middleware)
```

```text
Internet
    ↓
Nginx            ← reverse proxy: TLS, static files, load balancing
    ↓  plain HTTP over the private network
Uvicorn/Gunicorn ← application server: runs your Python
    ↓
Your FastAPI app
```

---

## 3. Why two layers

> [!IMPORTANT]
> Application servers are optimised for **running your code**, not for handling thousands of slow client connections, TLS handshakes and static files. A reverse proxy is written in C for exactly that, and it shields your workers from clients on poor mobile connections.

Without a reverse proxy, one slow client can occupy a worker process for seconds while it trickles in a request body. Nginx buffers the whole request first and only then hands a complete one to the application.

---

## 4. What a reverse proxy gives you

- **TLS termination** — certificates in one place
- **Static file serving** — orders of magnitude faster than through your application
- **Compression** — gzip/brotli without touching application code
- **Load balancing** across multiple application instances
- **Rate limiting and connection limits**
- **Buffering** — protects workers from slow clients
- **Caching** of responses
- **Health checks** and graceful removal of failing backends

---

## 5. The interface between the layers

Each ecosystem defines how a server hands a request to an application:

| Ecosystem | Interface | Servers |
|---|---|---|
| **Python (sync)** | WSGI | Gunicorn, uWSGI |
| **Python (async)** | ASGI | [Uvicorn](Uvicorn.md), Hypercorn |
| **Java** | Servlet API | Tomcat, Jetty, Undertow |
| **.NET** | Middleware pipeline | Kestrel |
| **Node.js** | The runtime *is* the server | http module, Express |
| **PHP** | FastCGI | PHP-FPM |
| **Go** | Built in | `net/http` |

> [!TIP]
> Go and Node.js include a production-grade HTTP server in the standard library, which is why they often run without a separate application server. They still usually sit behind a reverse proxy for TLS and static files.

---

## 6. Real World Example

- **The standard Python production stack** is Nginx → Gunicorn → Uvicorn workers → FastAPI.
- **Kubernetes** replaces the reverse proxy role with an Ingress controller, which is usually Nginx or Envoy underneath.
- **Cloud load balancers** (ALB, Cloud Load Balancing) take over TLS and load balancing, sometimes removing the need for your own Nginx layer entirely.

---

## 7. Where TLS terminates

```text
Option A   Cloud load balancer terminates TLS → plain HTTP inside the VPC
Option B   Nginx terminates TLS → plain HTTP to app servers
Option C   End-to-end TLS all the way to the application (zero-trust)
```

> [!CAUTION]
> With A or B, traffic inside your network is **plaintext**. That was acceptable when the internal network was considered trusted; zero-trust architecture assumes it is not, which is why internal TLS and mTLS are increasingly standard.

---

## 8. When To Use / When NOT To Use

> [!TIP]
> Use a reverse proxy in front of any production application. Even when a cloud load balancer handles TLS, having a proxy layer gives you caching, buffering and routing control.

> [!CAUTION]
> Do not serve static files through your application framework in production, and do not expose an application server directly to the internet. Development servers (`flask run`, `python manage.py runserver`, `next dev`) are explicitly not production-safe — they are single-threaded, unhardened and slow.

---

## 9. Mental Model

> [!NOTE]
> **The reverse proxy is the receptionist; the application server is the specialist.**
>
> The receptionist deals with everyone who walks in, handles the routine requests themselves, checks identity, and only passes on the ones that genuinely need the specialist — who is expensive and should not spend their day answering the door.

---

## 10. Key Takeaway

> [!IMPORTANT]
> A reverse proxy handles connections, TLS and static content; an application server runs your code — keep the roles separate and never expose the application server directly.

---

## 11. Common Mistakes

- **Running a development server in production**
- **Serving static files through the application**
- **Exposing the application server directly** to the internet
- **Forgetting `X-Forwarded-For`/`X-Forwarded-Proto`**, so the app sees the proxy's IP and thinks it is on HTTP
- **Mismatched timeouts** between proxy and application, producing confusing 502s
- **No request size limits** at the proxy

---

## 12. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 13. Workbook Exercise

- [ ] Draw your own deployment and label which component performs each role in section 4.
- [ ] Check whether your application receives the real client IP or the proxy's.
- [ ] Compare your proxy timeout with your application timeout and decide which should be longer.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Internet
    ↓
Reverse proxy (Nginx) — TLS, static, load balancing
    ↓
Application server (Uvicorn/Gunicorn/Kestrel)
    ↓
Your application
```

## 2. Request Flow

```text
Input       an HTTPS connection from a client
    ↓
Processing  TLS terminated, request buffered, routed to a healthy app instance
    ↓
Output      a response, compressed and possibly cached at the proxy
```

## 3. Real-World Usage

**Nginx in front of Gunicorn** is the most common Python production deployment in existence. The division of labour — C-level connection handling in front, Python application logic behind — is what makes it reliable under load.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Software that accepts HTTP connections, in two distinct roles |
| **Why does it exist?** | Because connection handling and application execution are different jobs |
| **Where does it belong?** | Between the internet and your application code |
| **When should I use it?** | Always in production — never expose an app server directly |

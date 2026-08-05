# Nginx

> **In one line —** an extremely fast reverse proxy and web server that handles tens of thousands of connections per worker, because it never blocks.

| | |
|---|---|
| **Category** | Reverse Proxy / Web Server |
| **Architectural Layer** | Edge, in front of the application |
| **Written in** | C |
| **Related notes** | [Web Servers](Web%20Servers.md) · [Load Balancing](../14%20-%20Scalability%20and%20Reliability/Load%20Balancing.md) · [TLS SSL](../04%20-%20Networking%20and%20Internet/TLS%20SSL.md) · [Caching](../10%20-%20Distributed%20Systems/Caching.md) |

---

## 1. Short Definition

*What is it?*

Nginx is a web server and reverse proxy that sits in front of application servers. It terminates TLS, serves static files, load-balances, compresses, caches and rate-limits — before your application sees anything.

---

## 2. Purpose

*What is its main purpose?*

To handle the parts of HTTP that have nothing to do with your business logic, and to do them far more efficiently than an application framework can.

---

## 3. Problem

*What engineering problem does it solve?*

The classic Apache model used one process or thread per connection. At 10,000 concurrent connections that meant 10,000 threads and gigabytes of memory — the **C10K problem**.

```text
THREAD PER CONNECTION            NGINX EVENT-DRIVEN
10,000 threads                   4 worker processes
~10 GB of stacks                 ~10 MB
constant context switching       an event loop per worker
```

Nginx applies the same [event loop](../05%20-%20Frontend%20Architecture/Event%20Loop.md) idea as Node.js, but in C, purely for connections.

---

## 4. Architecture Position

```text
Internet
    ↓  HTTPS
NGINX                    ← TLS, static files, compression, cache, rate limits
    ↓  plain HTTP over the private network
Application servers      (Uvicorn, Gunicorn, Kestrel, Node)
    ↓
Database / cache
```

---

## 5. What it does, in order

```text
1. Accept the TCP connection
2. Terminate TLS
3. Match the request against server/location blocks
4. Is it a static file?     → serve directly from disk (very fast)
5. Is it in the proxy cache? → serve from cache
6. Otherwise → choose a healthy upstream and proxy the request
7. Buffer the response, compress it, add headers
8. Return it to the client
```

> [!IMPORTANT]
> Step 7 matters more than it looks. Nginx **buffers** the full request from a slow client before forwarding it, so your application worker is never held open by someone on a bad mobile connection.

---

## 6. Core configuration concepts

```nginx
upstream app {
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;
}

server {
    listen 443 ssl;
    server_name example.com;

    location /static/ {
        root /var/www;              # served directly, never touches the app
        expires 1y;
    }

    location / {
        proxy_pass http://app;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 30s;
    }
}
```

> [!CAUTION]
> Those two `X-Forwarded-*` headers are not optional. Without them your application sees the proxy's IP as the client (breaking rate limiting and logging) and believes the connection is plain HTTP (breaking redirect and cookie logic).

---

## 7. Real World Example

- **Roughly a third of all websites** run Nginx, either directly or as part of a platform.
- **Kubernetes Ingress** is most commonly the Nginx Ingress Controller.
- **CDNs and API gateways** are frequently Nginx or its derivatives (OpenResty, Kong) underneath.

---

## 8. Input, Processing, Output

**Input:** HTTP/HTTPS connections, plus its configuration files.
**Processing:** event-driven, non-blocking connection handling across a small number of worker processes.
**Output:** a response served from disk, from cache, or proxied from an upstream — compressed and with the headers you configured.

---

## 9. Communication and Dependencies

- **Clients** in front
- **Application servers** behind, over HTTP or a socket
- **The file system** for static assets and its cache
- **Certificates** from Let's Encrypt or elsewhere

---

## 10. Alternatives

```text
Nginx        fastest, huge install base, configuration is its own language
    ↓
Caddy        automatic HTTPS out of the box, far simpler config
    ↓
HAProxy      superior pure load balancing and health checking
    ↓
Envoy        service mesh oriented, dynamic configuration, observability
    ↓
Traefik      auto-discovers containers; popular with Docker
    ↓
Cloud LB     ALB, Cloud Load Balancing — managed, no config files
```

> [!TIP]
> For a small project, **Caddy** is worth considering purely because it obtains and renews certificates automatically with a three-line configuration.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Nginx in front of any production application server, for TLS, static assets, buffering and load balancing.

> [!CAUTION]
> Do not use it as a replacement for an application server, and do not put complex logic in its configuration. Nginx config is a declarative language that becomes very hard to test and reason about once it grows conditionals and rewrites.

---

## 12. Advantages and Disadvantages

**Advantages**
- Exceptional performance and very low memory use
- Static file serving far faster than any framework
- Mature, stable, extremely well documented
- Reloads configuration without dropping connections

**Disadvantages**
- Configuration syntax is idiosyncratic and easy to get subtly wrong
- Dynamic upstreams need extra modules or a rewrite-and-reload
- Debugging config errors is unpleasant
- Some features (active health checks, advanced load balancing) are commercial-only

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Connections** | Tens of thousands per worker |
| **Memory** | A few MB per worker |
| **CPU** | Minimal; TLS is the main cost and is hardware-accelerated |
| **Latency** | Sub-millisecond added |
| **Static files** | Often 10–100× faster than through an application |

---

## 14. Security Considerations

> [!CAUTION]
> Nginx is your **outermost defence**, and misconfiguration there exposes everything behind it. A `location` block that accidentally serves your application directory is a full source-code disclosure.

- **Set `client_max_body_size`** — the default allows large uploads that can exhaust disk or memory
- **Rate limit** with `limit_req` — the cheapest defence against brute force and scraping
- **Add security headers** centrally: HSTS, `X-Content-Type-Options`, CSP
- **Hide the version** (`server_tokens off`) to reduce fingerprinting
- **Sanitise `X-Forwarded-For`** — overwrite it rather than appending blindly, or clients can forge it
- **Keep it patched** — it is internet-facing C code

---

## 15. Mental Model

> [!NOTE]
> **Nginx is the doorman of a building.**
>
> They check identity, hand out the leaflets everyone asks for without disturbing anyone upstairs, turn away obvious troublemakers, and only send genuine visitors up — and they do all of it for thousands of people without ever leaving the door.

---

## 16. Mini Architecture Diagram

```text
Clients
    ↓
┌──────────── NGINX ────────────┐
│  TLS termination              │
│  static files                 │
│  gzip / brotli                │
│  proxy cache                  │
│  rate limiting                │
│  load balancing               │
└───────┬───────────────┬───────┘
        ↓               ↓
   App server 1    App server 2
```

---

## 17. Complete Request Flow

```text
HTTPS request arrives
    ↓
TLS terminated
    ↓
location match:  /static/  → served from disk, request never reaches the app
    ↓  otherwise
Rate limit check → 429 if exceeded
    ↓
Proxy cache hit? → return cached response
    ↓ miss
Pick a healthy upstream (round robin / least connections)
    ↓
Request fully buffered, then forwarded
    ↓
Response received, compressed, cached if allowed
    ↓
Returned to the client; the connection is kept alive for reuse
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Nginx handles connections, TLS and static content with an event loop in C — putting it in front of your application removes an enormous amount of work your framework should never have been doing.

---

## 19. Common Mistakes

- **Forgetting `X-Forwarded-For` and `X-Forwarded-Proto`**
- **Leaving `client_max_body_size` at the default** and getting mysterious 413s — or accepting huge uploads
- **Mismatched timeouts** between Nginx and the application, producing 502s
- **Complex rewrite logic** that nobody can test
- **Serving static files through the app** while Nginx sits idle in front
- **Not reloading after a certificate renewal**

---

## 20. Open Source Technologies

- **Nginx** — the server itself
- **Caddy** — automatic HTTPS, much simpler configuration
- **HAProxy** — specialised load balancing
- **Envoy**, **Traefik** — dynamic, container-native proxies
- **OpenResty** — Nginx with Lua scripting
- **certbot** — certificate automation

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Add rate limiting to one endpoint and verify it returns 429 under load.
- [ ] Check that your application logs the real client IP rather than the proxy's.
- [ ] Move static file serving from your application to Nginx and compare response times.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Internet
    ↓
Nginx
    ↓
Application servers
    ↓
Database
```

## 2. Request Flow

```text
Input       an HTTPS connection
    ↓
Processing  TLS → routing → static/cache/proxy → compression
    ↓
Output      a response, with your application involved only when necessary
```

## 3. Real-World Usage

**Kubernetes Ingress** is most often the Nginx Ingress Controller: the same software that has served static files for two decades now routes traffic into containerised microservices. The role has not changed, only what sits behind it.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An event-driven reverse proxy and web server |
| **Why does it exist?** | Because thread-per-connection did not scale, and app servers should not face clients |
| **Where does it belong?** | Between the internet and your application servers |
| **When should I use it?** | In front of essentially any production backend |

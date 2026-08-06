# The Complete System Map

> **In one line —** one journey, from a user's click to the product in their hands, told as a sequence of stations; at each station, what happens, which concepts live there, and what can go wrong.

| | |
|---|---|
| **Category** | Map *(the whole handbook on one page)* |
| **Read it** | Before the handbook, to see the shape. Again after, to connect it. |
| **Related notes** | [How To Use This Handbook](How%20To%20Use%20This%20Handbook.md) · [README](README.md) · [Architect Mindset](Architect%20Mindset.md) · [Diagrams](../17%20-%20Personal%20Knowledge%20Base/Diagrams.md) |

---

## How to use this map

This is not a lesson. It contains **no definitions** — every definition is one click away, in the note linked at that station.

```text
YOU ARE HERE FOR ONE OF TWO REASONS

    "I want to see the whole thing at once"
        → read MAP 1, the one-screen version. Stop there. That is enough.

    "I am working on X and I do not know where it sits"
        → find X in the station index at the bottom
        → read that station, then follow the link
```

> [!IMPORTANT]
> **The handbook's 204 notes are the *what*; this map is the *where*.** Almost everything confusing about software architecture is not the individual pieces — it is not knowing which piece runs before which, who is waiting for whom, and which layer a problem actually lives in. That is the only thing this file is for.

> [!TIP]
> **Read MAP 5 — the failure map — last, and then read it again in six months.** Knowing the path is a beginner's skill. Knowing what waits on the path at each station is what separates someone who can build a system from someone who can keep one running.

---

# MAP 1 — THE JOURNEY

**One click. One product. Everything in between.**

```text
                        ┌──────────────────┐
                        │    THE USER      │   clicks "Buy"
                        └────────┬─────────┘
                                 │
  ══════════ THEIR DEVICE ═══════▼══════════════════════════════
   01   the click becomes an event, JavaScript runs
   02   DNS: a name becomes an address
   03   TCP handshake, then TLS handshake
   04   an HTTP request is built and sent

  ══════════ THE NETWORK ════════▼══════════════════════════════
   05   router → ISP → across the internet
   06   CDN / edge: if it is cached, IT STOPS HERE  ──────┐
                                                          │
  ══════════ YOUR CLOUD ═════════▼═══════════════════════ │ ═══
   07   region, and the VPC boundary                      │
   08   load balancer chooses a healthy backend           │
   09   reverse proxy / web server                        │
                                                          │
  ══════════ YOUR APPLICATION ═══▼═══════════════════════ │ ═══
   10   runtime: a process, a thread, an event loop       │
   11   the middleware chain                              │
   12   routing → controller                              │
   13   WHO ARE YOU?        authentication                 │
   14   MAY YOU DO THIS?    authorisation                  │
   15   IS THIS VALID?      validation                     │
   16   the business logic — the only part that is YOURS   │
                                                          │
  ══════════ YOUR DATA ══════════▼═══════════════════════ │ ═══
   17   cache: if it is a hit, IT STOPS HERE  ──────────┐ │
   18   repository / ORM                                │ │
   19   the database: SQL, an index, a transaction       │ │
   20   ▼▼▼ THE FLOOR: CPU, RAM, disk, kernel ▼▼▼       │ │
                                                        │ │
  ══════════ WORK TOO SLOW FOR A REQUEST ══════════════ │ │ ══
   21   queue → worker    THE RESPONSE DOES NOT WAIT    │ │
        (email, transcode, AI, reports, webhooks)       │ │
                                                        │ │
  ══════════ BACK OUT ═══════════▲═════════════════════ │ │ ══
   22   response serialised into JSON / HTML            │ │
   23   back out through proxy → LB → edge → network ◄──┴─┘
   24   the browser parses, renders, paints

                        ┌────────▼─────────┐
                        │   THE PRODUCT    │   the user sees it
                        └──────────────────┘

        Stations 06 and 17 are the two SHORTCUTS.
        A well-built system answers most requests at one of them
        and never reaches the database at all.
```

> [!IMPORTANT]
> **Notice the two exits marked "IT STOPS HERE".** The whole of performance engineering is the effort to end requests as early as possible on that column. A request answered at the edge costs nothing; the same request answered at station 19 costs a database. When someone says "we added a CDN and a cache", what they mean is that most journeys now stop at 06 or 17.

---

# THE STATIONS

## ZONE A — Their device

### 01 · The click becomes an event

**WHAT HAPPENS** — The browser turns a physical click into an event, hands it to JavaScript, and your code decides to make a request. Nothing has left the machine yet.

**CONCEPTS** — [Browser Internals](../05%20-%20Frontend%20Architecture/Browser%20Internals.md) · [Event Loop](../05%20-%20Frontend%20Architecture/Event%20Loop.md) · [JavaScript Engine](../05%20-%20Frontend%20Architecture/JavaScript%20Engine.md) · [DOM and CSSOM](../05%20-%20Frontend%20Architecture/DOM%20and%20CSSOM.md) · [JavaScript](../05%20-%20Frontend%20Architecture/JavaScript.md) · [State Management](../05%20-%20Frontend%20Architecture/State%20Management.md)

**WHAT CAN GO WRONG** — blocking the event loop, so the page freezes before the request is even sent · no loading state, so the user clicks three times · no client-side timeout · state updated optimistically and never reconciled

---

### 02 · DNS: a name becomes an address

**WHAT HAPPENS** — `shop.example.com` means nothing to the network. It must become an IP address, via a chain of caches: browser → OS → resolver → authoritative server.

**CONCEPTS** — [DNS](../04%20-%20Networking%20and%20Internet/DNS.md) · [IP Addresses](../04%20-%20Networking%20and%20Internet/IP%20Addresses.md) · [Internet Fundamentals](../04%20-%20Networking%20and%20Internet/Internet%20Fundamentals.md)

**WHAT CAN GO WRONG** — a TTL of hours, so failover takes hours too · a client library that resolves once and caches forever · DNS itself being the outage, while every other metric looks healthy

---

### 03 · TCP handshake, then TLS handshake

**WHAT HAPPENS** — A connection is established (TCP), then encrypted (TLS): certificate presented, verified, keys agreed. Two round trips before a single byte of your request moves.

**CONCEPTS** — [TCP IP](../04%20-%20Networking%20and%20Internet/TCP%20IP.md) · [TLS SSL](../04%20-%20Networking%20and%20Internet/TLS%20SSL.md) · [ISP Router Switch Modem](../04%20-%20Networking%20and%20Internet/ISP%20Router%20Switch%20Modem.md)

**WHAT CAN GO WRONG** — an expired certificate, which is one of the most common self-inflicted outages there is · no connection reuse, paying the handshake on every request · a cross-continent handshake costing hundreds of milliseconds before anything useful happens

---

### 04 · The HTTP request is built

**WHAT HAPPENS** — Method, path, headers, cookies, body. The cookie carrying the session is attached here, automatically, by the browser — which is why CSRF exists as a category.

**CONCEPTS** — [HTTP HTTPS](../04%20-%20Networking%20and%20Internet/HTTP%20HTTPS.md) · [API Design](../04%20-%20Networking%20and%20Internet/API%20Design.md) · [WebSockets](../04%20-%20Networking%20and%20Internet/WebSockets.md) · [gRPC](../04%20-%20Networking%20and%20Internet/gRPC.md)

**WHAT CAN GO WRONG** — a session token in `localStorage` instead of an `HttpOnly` cookie, so any XSS is account takeover · no idempotency key on a request that spends money · an API shape that forces the client into twenty round trips

---

## ZONE B — The network

### 05 · Router → ISP → the internet

**WHAT HAPPENS** — Packets leave the building and cross networks you do not own and cannot see. Latency here is bounded by physics, not by engineering.

**CONCEPTS** — [Internet Fundamentals](../04%20-%20Networking%20and%20Internet/Internet%20Fundamentals.md) · [ISP Router Switch Modem](../04%20-%20Networking%20and%20Internet/ISP%20Router%20Switch%20Modem.md) · [IP Addresses](../04%20-%20Networking%20and%20Internet/IP%20Addresses.md)

**WHAT CAN GO WRONG** — mobile networks drop connections constantly, so anything long-running needs resumability · packet loss appears as inexplicable slowness · you cannot fix this layer, only design around it

---

### 06 · CDN / edge — **the first shortcut**

**WHAT HAPPENS** — A cache close to the user. If the answer is here, the journey ends: no cloud, no application, no database. Immutable things — images, video segments, built JavaScript — should almost always end here.

**CONCEPTS** — [Internet Fundamentals](../04%20-%20Networking%20and%20Internet/Internet%20Fundamentals.md) · [Cloud Fundamentals](../12%20-%20Cloud%20Architecture/Cloud%20Fundamentals.md) · [Load Balancing](../14%20-%20Scalability%20and%20Reliability/Load%20Balancing.md) · [Netflix Architecture](../15%20-%20Real%20World%20System%20Design/Netflix%20Architecture.md)

**WHAT CAN GO WRONG** — nothing cached, so every request travels the whole path and you pay full egress · caching a personalised page and serving one user's data to another · cache lifetime on a manifest set as long as on a segment, so nothing can ever be updated

---

## ZONE C — Your cloud

### 07 · The region and the VPC boundary

**WHAT HAPPENS** — The request enters infrastructure you rent. A route table decides whether it may go further. Everything from here on is inside a network you defined.

**CONCEPTS** — [VPC](../12%20-%20Cloud%20Architecture/VPC.md) · [Cloud Fundamentals](../12%20-%20Cloud%20Architecture/Cloud%20Fundamentals.md) · [AWS Architecture](../12%20-%20Cloud%20Architecture/AWS%20Architecture.md) · [IaaS PaaS SaaS](../12%20-%20Cloud%20Architecture/IaaS%20PaaS%20SaaS.md) · [Cloud Security](../12%20-%20Cloud%20Architecture/Cloud%20Security.md)

**WHAT CAN GO WRONG** — a database in a public subnet, reachable from the internet · a security group open to `0.0.0.0/0` · everything in one availability zone · a NAT gateway quietly becoming a top-three line item

---

### 08 · The load balancer

**WHAT HAPPENS** — TLS is terminated, a *healthy* backend is chosen, the request is forwarded. Its most valuable job is not balancing — it is noticing which backend is broken and routing around it.

**CONCEPTS** — [Load Balancing](../14%20-%20Scalability%20and%20Reliability/Load%20Balancing.md) · [High Availability](../14%20-%20Scalability%20and%20Reliability/High%20Availability.md) · [Horizontal Scaling](../14%20-%20Scalability%20and%20Reliability/Horizontal%20Scaling.md) · [TLS SSL](../04%20-%20Networking%20and%20Internet/TLS%20SSL.md)

**WHAT CAN GO WRONG** — a deep health check that removes *every* backend when one database is slow, turning a degradation into an outage · an idle timeout that silently kills a long report · draining shorter than the slowest request, dropping requests on every deployment · retries amplifying an overload into a collapse

---

### 09 · Reverse proxy / web server

**WHAT HAPPENS** — Static files served directly, compression applied, the request handed to your application process over a socket.

**CONCEPTS** — [Nginx](../06%20-%20Backend%20Architecture/Nginx.md) · [Web Servers](../06%20-%20Backend%20Architecture/Web%20Servers.md) · [Gunicorn](../06%20-%20Backend%20Architecture/Gunicorn.md) · [Uvicorn](../06%20-%20Backend%20Architecture/Uvicorn.md)

**WHAT CAN GO WRONG** — a body size limit rejecting uploads with an unhelpful error · a proxy timeout shorter than the application's · too few worker processes, so requests queue invisibly before your code ever runs

---

## ZONE D — Your application

### 10 · The runtime: a process, a thread, an event loop

**WHAT HAPPENS** — Your code is not magic; it is a process the operating system schedules. Whether the next request waits depends entirely on this layer's concurrency model.

**CONCEPTS** — [Runtime Explained](../06%20-%20Backend%20Architecture/Runtime%20Explained.md) · [Process](../02%20-%20Computer%20Science%20Fundamentals/Process.md) · [Thread](../02%20-%20Computer%20Science%20Fundamentals/Thread.md) · [Multithreading](../02%20-%20Computer%20Science%20Fundamentals/Multithreading.md) · [Node.js Runtime](../03%20-%20Programming%20Languages%20and%20Runtime/Node.js%20Runtime.md) · [JVM](../03%20-%20Programming%20Languages%20and%20Runtime/JVM.md) · [CPython](../03%20-%20Programming%20Languages%20and%20Runtime/CPython.md) · [Bytecode](../03%20-%20Programming%20Languages%20and%20Runtime/Bytecode.md) · [Compiled vs Interpreted Languages](../03%20-%20Programming%20Languages%20and%20Runtime/Compiled%20vs%20Interpreted%20Languages.md)

**WHAT CAN GO WRONG** — one blocking call holding the event loop, so the whole service stalls · a thread pool exhausted by a slow dependency, with nothing left for healthy requests · a memory leak that only appears after six hours

---

### 11 · The middleware chain

**WHAT HAPPENS** — Every request passes through the same ordered pipeline before reaching your handler: logging, request id, CORS, body parsing, rate limits, error handling.

**CONCEPTS** — [Middleware](../07%20-%20Backend%20Design%20Patterns/Middleware.md) · [Backend Fundamentals](../06%20-%20Backend%20Architecture/Backend%20Fundamentals.md) · [Backend Frameworks](../06%20-%20Backend%20Architecture/Backend%20Frameworks.md)

**WHAT CAN GO WRONG** — order matters and it is easy to get wrong: authentication after logging means credentials in your logs · an error handler registered before the routes, so it never catches anything · no request id generated here, which makes every later investigation manual

---

### 12 · Routing → controller

**WHAT HAPPENS** — The path and method are matched to one function. That function's only job is to translate HTTP into a call to your business logic, and back.

**CONCEPTS** — [Routing](../07%20-%20Backend%20Design%20Patterns/Routing.md) · [Controllers](../07%20-%20Backend%20Design%20Patterns/Controllers.md) · [API Design](../04%20-%20Networking%20and%20Internet/API%20Design.md) · [Express.js](../06%20-%20Backend%20Architecture/Express.js.md) · [FastAPI](../06%20-%20Backend%20Architecture/FastAPI.md) · [NestJS](../06%20-%20Backend%20Architecture/NestJS.md) · [Django](../06%20-%20Backend%20Architecture/Django.md) · [Spring Boot](../06%20-%20Backend%20Architecture/Spring%20Boot.md) · [ASP.NET Core](../06%20-%20Backend%20Architecture/ASP.NET%20Core.md) · [Laravel](../06%20-%20Backend%20Architecture/Laravel.md)

**WHAT CAN GO WRONG** — business logic written inside the controller, so it cannot be tested or reused · database queries in the controller · a fat controller that becomes the file nobody wants to open

---

### 13 · WHO ARE YOU? — authentication

**WHAT HAPPENS** — The session token or JWT is verified. The request stops being anonymous. Note the ordering: identity is established *before* anything is permitted.

**CONCEPTS** — [Authentication](../09%20-%20Security/Authentication.md) · [JWT](../09%20-%20Security/JWT.md) · [Session Store](../08%20-%20Databases%20and%20Data/Session%20Store.md) · [OAuth2](../09%20-%20Security/OAuth2.md) · [OpenID Connect](../09%20-%20Security/OpenID%20Connect.md) · [Password Hashing](../09%20-%20Security/Password%20Hashing.md) · [Argon2](../09%20-%20Security/Argon2.md) · [Login System Architecture](../15%20-%20Real%20World%20System%20Design/Login%20System%20Architecture.md)

**WHAT CAN GO WRONG** — a long-lived stateless JWT, so logout does not actually work · a fast hash on passwords, so a leaked table is an immediate mass takeover · the session store being a single node, so losing it logs everyone out

---

### 14 · MAY YOU DO THIS? — authorisation

**WHAT HAPPENS** — A different question from the last one, and the one more often forgotten: this user is authenticated, but may they read *this particular record*?

**CONCEPTS** — [Authorization](../09%20-%20Security/Authorization.md) · [IAM](../12%20-%20Cloud%20Architecture/IAM.md) · [OWASP Top 10](../09%20-%20Security/OWASP%20Top%2010.md) · [Security Checklist](../16%20-%20Templates/Security%20Checklist.md)

**WHAT CAN GO WRONG** — **the single most common serious application flaw: checking authentication and forgetting authorisation**, so changing an id in the URL returns someone else's data · permission checked in the UI but not on the server · a cache key without the tenant, leaking data between customers

---

### 15 · IS THIS VALID? — validation

**WHAT HAPPENS** — Everything from the client is untrusted, including from your own frontend. Types, ranges, formats and business rules are checked here, server-side, before any of it reaches your logic.

**CONCEPTS** — [Validation](../07%20-%20Backend%20Design%20Patterns/Validation.md) · [Secure Coding](../09%20-%20Security/Secure%20Coding.md) · [OWASP Top 10](../09%20-%20Security/OWASP%20Top%2010.md) · [TypeScript](../05%20-%20Frontend%20Architecture/TypeScript.md)

**WHAT CAN GO WRONG** — trusting a price, total or role sent by the client · SQL built by string concatenation · a URL the server will fetch without an allowlist (SSRF) · an uploaded file trusted by its extension

---

### 16 · The business logic — the only part that is yours

**WHAT HAPPENS** — Everything up to here is the same in every application in the world. This station is the product. It is also the only station where "what should happen" cannot be looked up.

**CONCEPTS** — [Services](../07%20-%20Backend%20Design%20Patterns/Services.md) · [Clean Architecture](../07%20-%20Backend%20Design%20Patterns/Clean%20Architecture.md) · [Domain Driven Design](../07%20-%20Backend%20Design%20Patterns/Domain%20Driven%20Design.md) · [SOLID Principles](../07%20-%20Backend%20Design%20Patterns/SOLID%20Principles.md) · [Design Patterns](../07%20-%20Backend%20Design%20Patterns/Design%20Patterns.md) · [Dependency Injection](../07%20-%20Backend%20Design%20Patterns/Dependency%20Injection.md) · [Monolith](../10%20-%20Distributed%20Systems/Monolith.md) · [Modular Monolith](../10%20-%20Distributed%20Systems/Modular%20Monolith.md) · [Microservices](../10%20-%20Distributed%20Systems/Microservices.md)

**WHAT CAN GO WRONG** — logic spread so thinly across layers that no single file explains a rule · a call to another service with no timeout, so its bad day is your outage · a multi-step process with no compensation, leaving records half-finished · split into microservices before the team was large enough to need it

---

## ZONE E — Your data

### 17 · The cache — **the second shortcut**

**WHAT HAPPENS** — Memory is roughly a hundred times faster than disk, and most applications read the same small set of data repeatedly. If the answer is here, the journey ends.

**CONCEPTS** — [Cache](../08%20-%20Databases%20and%20Data/Cache.md) · [Redis](../08%20-%20Databases%20and%20Data/Redis.md) · [Caching](../10%20-%20Distributed%20Systems/Caching.md) · [Cache Memory](../02%20-%20Computer%20Science%20Fundamentals/Cache%20Memory.md) · [Session Store](../08%20-%20Databases%20and%20Data/Session%20Store.md)

**WHAT CAN GO WRONG** — **a cache key that omits the user or tenant — a data leak dressed as an optimisation** · caching used to hide a query that needed an index · no invalidation strategy, so users see stale data indefinitely · the cache treated as durable, and it is not

---

### 18 · Repository / ORM

**WHAT HAPPENS** — Objects become SQL. Convenient, and the layer where the most common performance defect in the industry is created without anyone noticing.

**CONCEPTS** — [Repository Pattern](../07%20-%20Backend%20Design%20Patterns/Repository%20Pattern.md) · [ORM](../08%20-%20Databases%20and%20Data/ORM.md) · [Prisma](../08%20-%20Databases%20and%20Data/Prisma.md) · [SQLAlchemy](../08%20-%20Databases%20and%20Data/SQLAlchemy.md) · [Entity Framework](../08%20-%20Databases%20and%20Data/Entity%20Framework.md) · [Hibernate](../08%20-%20Databases%20and%20Data/Hibernate.md)

**WHAT CAN GO WRONG** — **the N+1 query: one query, then one more per row, four hundred round trips for one page** · a lazy-loaded association inside a loop · `SELECT *` dragging large text columns across the network · an unbounded query that worked until the table grew

---

### 19 · The database

**WHAT HAPPENS** — The source of truth. A query is planned, an index is used or ignored, a transaction commits. Almost always the slowest station, and almost always the real bottleneck.

**CONCEPTS** — [Database Fundamentals](../08%20-%20Databases%20and%20Data/Database%20Fundamentals.md) · [PostgreSQL](../08%20-%20Databases%20and%20Data/PostgreSQL.md) · [SQL](../08%20-%20Databases%20and%20Data/SQL.md) · [Indexes](../08%20-%20Databases%20and%20Data/Indexes.md) · [Transactions](../08%20-%20Databases%20and%20Data/Transactions.md) · [Database Optimization](../08%20-%20Databases%20and%20Data/Database%20Optimization.md) · [MySQL](../08%20-%20Databases%20and%20Data/MySQL.md) · [MongoDB](../08%20-%20Databases%20and%20Data/MongoDB.md) · [RDS](../12%20-%20Cloud%20Architecture/RDS.md) · [Distributed Lock](../08%20-%20Databases%20and%20Data/Distributed%20Lock.md)

**WHAT CAN GO WRONG** — a missing index, turning 40 ms into 4 seconds · connection pool exhausted, so a healthy database looks down · a long transaction holding locks and blocking everything behind it · a single primary that no amount of extra application servers can help

---

### 20 · ▼ THE FLOOR — what every station above stands on

**WHAT HAPPENS** — Nothing here is a station on the journey. This is the ground *underneath all twenty-four of them*. Every request, at every layer, is ultimately a process using a CPU, memory and a disk, mediated by a kernel.

**CONCEPTS** — [How Computers Work](../02%20-%20Computer%20Science%20Fundamentals/01%20-%20How%20Computers%20Work.md) · [Operating Systems](../02%20-%20Computer%20Science%20Fundamentals/02%20-%20Operating%20Systems.md) · [Processes and Threads](../02%20-%20Computer%20Science%20Fundamentals/03%20-%20Processes%20and%20Threads.md) · [CPU](../02%20-%20Computer%20Science%20Fundamentals/CPU.md) · [RAM](../02%20-%20Computer%20Science%20Fundamentals/RAM.md) · [SSD and HDD](../02%20-%20Computer%20Science%20Fundamentals/SSD%20and%20HDD.md) · [Kernel](../02%20-%20Computer%20Science%20Fundamentals/Kernel.md) · [System Calls](../02%20-%20Computer%20Science%20Fundamentals/System%20Calls.md) · [Memory Management](../02%20-%20Computer%20Science%20Fundamentals/Memory%20Management.md) · [Scheduler](../02%20-%20Computer%20Science%20Fundamentals/Scheduler.md) · [Context Switching](../02%20-%20Computer%20Science%20Fundamentals/Context%20Switching.md) · [File Systems](../02%20-%20Computer%20Science%20Fundamentals/File%20Systems.md) · [User Space](../02%20-%20Computer%20Science%20Fundamentals/User%20Space.md) · [Registers](../02%20-%20Computer%20Science%20Fundamentals/Registers.md) · [Binary and Machine Code](../02%20-%20Computer%20Science%20Fundamentals/Binary%20and%20Machine%20Code.md)

**WHY IT MATTERS** — This is the layer that explains *why* the advice above is true. Caching works because of the RAM-to-disk gap. An event loop stalls because of how the scheduler works. A container is light because it shares a kernel. You can build systems without this layer; you cannot debug the strange ones.

---

## ZONE F — Work too slow for a request

### 21 · Queue → worker — **the response does not wait**

**WHAT HAPPENS** — Anything slow is written to a queue and answered later: emails, transcoding, reports, AI generation, webhooks, invoices. The user gets a response in milliseconds; the work happens behind them.

**CONCEPTS** — [Message Queues](../10%20-%20Distributed%20Systems/Message%20Queues.md) · [Background Workers](../10%20-%20Distributed%20Systems/Background%20Workers.md) · [Event Driven Architecture](../10%20-%20Distributed%20Systems/Event%20Driven%20Architecture.md) · [Kafka](../10%20-%20Distributed%20Systems/Kafka.md) · [RabbitMQ](../10%20-%20Distributed%20Systems/RabbitMQ.md) · [AWS SQS](../10%20-%20Distributed%20Systems/AWS%20SQS.md) · [Pub Sub](../08%20-%20Databases%20and%20Data/Pub%20Sub.md) · [Redis Streams](../10%20-%20Distributed%20Systems/Redis%20Streams.md) · [Celery](../10%20-%20Distributed%20Systems/Celery.md) · [BullMQ](../10%20-%20Distributed%20Systems/BullMQ.md) · [Hangfire](../10%20-%20Distributed%20Systems/Hangfire.md) · [Serverless](../10%20-%20Distributed%20Systems/Serverless.md) · [Lambda](../12%20-%20Cloud%20Architecture/Lambda.md)

**WHAT CAN GO WRONG** — a message delivered twice and the customer charged twice, because the handler was not idempotent · no dead letter queue, so failures vanish silently · a visibility timeout shorter than the job, so two workers do the same work · queue depth unmonitored, so a stalled fleet is invisible until customers complain

> [!IMPORTANT]
> **This station is the single biggest lever on perceived speed in most systems.** Moving work off the request path converts a capacity problem into a latency-of-completion problem, which is almost always cheaper. It is also where correctness gets hard: the moment work is retried, everything it touches must be idempotent.

---

## ZONE G — Back out

### 22 · The response is assembled

**WHAT HAPPENS** — Objects become JSON or HTML, headers are set, the status code is chosen. Small decisions here shape every client that ever consumes it.

**CONCEPTS** — [API Design](../04%20-%20Networking%20and%20Internet/API%20Design.md) · [HTTP HTTPS](../04%20-%20Networking%20and%20Internet/HTTP%20HTTPS.md) · [Next.js](../05%20-%20Frontend%20Architecture/Next.js.md) · [gRPC](../04%20-%20Networking%20and%20Internet/gRPC.md)

**WHAT CAN GO WRONG** — a stack trace or internal version returned in an error · returning 200 with an error inside the body, so nothing can alert on it · an unpaginated list that grows with your success · sensitive fields serialised because the model was returned directly

---

### 23 · Back out through every layer

**WHAT HAPPENS** — The same path in reverse: proxy, load balancer, edge, network. Egress is metered on the way out — bytes leaving the cloud are the line item that surprises everyone.

**CONCEPTS** — [Cloud Fundamentals](../12%20-%20Cloud%20Architecture/Cloud%20Fundamentals.md) · [Load Balancing](../14%20-%20Scalability%20and%20Reliability/Load%20Balancing.md) · [S3](../12%20-%20Cloud%20Architecture/S3.md)

**WHAT CAN GO WRONG** — large files proxied through your application instead of handed off with a signed URL · no compression · egress paid at full rate because nothing sits in front of the bucket

---

### 24 · The browser renders it

**WHAT HAPPENS** — HTML parsed, CSS matched, layout calculated, pixels painted. Backend engineers routinely forget that the user's clock is still running here — and often for longer than everything above took.

**CONCEPTS** — [Rendering Engine](../05%20-%20Frontend%20Architecture/Rendering%20Engine.md) · [DOM and CSSOM](../05%20-%20Frontend%20Architecture/DOM%20and%20CSSOM.md) · [Browser Internals](../05%20-%20Frontend%20Architecture/Browser%20Internals.md) · [CSS](../05%20-%20Frontend%20Architecture/CSS.md) · [HTML](../05%20-%20Frontend%20Architecture/HTML.md) · [React](../05%20-%20Frontend%20Architecture/React.md) · [Next.js](../05%20-%20Frontend%20Architecture/Next.js.md) · [Frontend Architecture](../05%20-%20Frontend%20Architecture/Frontend%20Architecture.md)

**WHAT CAN GO WRONG** — a 45 ms API response inside a 4-second page load · unsized images causing layout shift · a blocking third-party script · a bundle so large that parsing it costs more than fetching it

> [!CAUTION]
> **A system is only as fast as station 24, and this is the most commonly ignored fact in backend engineering.** Optimising a request from 200 ms to 45 ms changes nothing a user can perceive if the page then takes four seconds to become usable. Measure what the user experiences, not what your service reports.

---

# MAP 2 — THE ASYNC PATH

**What happens after the user has already got their answer.**

```text
   station 21 dropped a message on a queue and the user left
                            │
                    ┌───────▼────────┐
                    │     QUEUE      │  depth + oldest-message age
                    │  (SQS, Kafka,  │  ← THE two metrics to alarm on
                    │   RabbitMQ)    │
                    └───────┬────────┘
                            │  workers scale on DEPTH, not on CPU
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   ┌─────────┐        ┌──────────┐        ┌──────────┐
   │ worker  │        │  worker  │        │  worker  │
   │ (spot)  │        │  (spot)  │        │  (spot)  │  scale to ZERO
   └────┬────┘        └────┬─────┘        └────┬─────┘  when idle
        │                  │                    │
        │  MUST BE IDEMPOTENT — it WILL run twice
        │                  │                    │
        ▼                  ▼                    ▼
   ┌──────────┐    ┌───────────────┐    ┌──────────────┐
   │  EMAIL   │    │  MEDIA        │    │  AI / ML     │
   │  SMS     │    │  transcode    │    │  embed       │
   │  webhook │    │  thumbnails   │    │  retrieve    │
   │  invoice │    │  → S3 → CDN   │    │  generate    │
   └──────────┘    └───────────────┘    └──────────────┘
        │                  │                    │
        └──────────────────┼────────────────────┘
                           ▼
                 write result to the database
                 notify the user (they may have closed the tab)
                           │
                 failed 3 times?
                           ▼
                 ┌──────────────────────┐
                 │  DEAD LETTER QUEUE   │ → alarm → a human
                 └──────────────────────┘
                 ← without this, failures disappear silently
```

**FULL CASE STUDIES OF THIS PATH** — [Video Processing Platform](../15%20-%20Real%20World%20System%20Design/Video%20Processing%20Platform.md) · [AI Application Architecture](../15%20-%20Real%20World%20System%20Design/AI%20Application%20Architecture.md) · [AI Pipelines](../11%20-%20AI%20Engineering/AI%20Pipelines.md) · [RAG](../11%20-%20AI%20Engineering/RAG.md) · [Model Serving](../11%20-%20AI%20Engineering/Model%20Serving.md) · [Vector Databases](../11%20-%20AI%20Engineering/Vector%20Databases.md) · [Embeddings](../11%20-%20AI%20Engineering/Embeddings.md)

> [!TIP]
> **Every box on this map is the same shape: accept fast, queue, process idempotently, store the result, notify.** Change the words and it is video transcoding, document conversion, AI generation or invoice rendering. Learn the shape once and all four are the same system.

---

# MAP 3 — HOW THE CODE GOT THERE

**None of MAP 1 exists until something puts it in production.**

```text
   an engineer's editor
        │
   ┌────▼──────────────────────────────────────────────┐
   │  GIT: a graph of immutable snapshots               │
   │  branch = a movable label                          │
   └────┬──────────────────────────────────────────────┘
        │ push a short-lived branch
   ┌────▼──────────────────────────────────────────────┐
   │  PULL REQUEST — the coordination point             │
   │    lint · types · unit tests · security scan       │
   │    build · integration tests   ← in PARALLEL       │
   │    human review  ← small diffs only get real review│
   └────┬──────────────────────────────────────────────┘
        │ all checks green, main is PROTECTED
   ┌────▼──────────────────────────────────────────────┐
   │  BUILD THE ARTEFACT — exactly ONCE                 │
   │  a container image tagged with the commit SHA      │
   │  → the SAME bytes go to staging AND production     │
   └────┬──────────────────────────────────────────────┘
        │ push to a registry, sign it
   ┌────▼──────────────────────────────────────────────┐
   │  INFRASTRUCTURE — declared, planned, reviewed      │
   │  Terraform: plan on the PR, apply on merge         │
   │  (the VPC, LB, database and cluster of MAP 1)      │
   └────┬──────────────────────────────────────────────┘
        │ OIDC — no stored cloud keys anywhere
   ┌────▼──────────────────────────────────────────────┐
   │  DEPLOY, reversibly                                │
   │  rolling / canary / blue-green                      │
   │  5% of traffic → compare metrics → promote or REVERT│
   │  the feature stays OFF behind a flag               │
   └────┬──────────────────────────────────────────────┘
        │
   ┌────▼──────────────────────────────────────────────┐
   │  IT IS NOW STATION 08-16 OF MAP 1                  │
   └───────────────────────────────────────────────────┘

   ROLLBACK is the whole point: seconds by flag, minutes by image.
   And the database migration is the ONE thing that does not roll back.
```

**CONCEPTS** — [Git](../13%20-%20DevOps%20and%20Delivery/Git.md) · [GitHub](../13%20-%20DevOps%20and%20Delivery/GitHub.md) · [CI CD](../13%20-%20DevOps%20and%20Delivery/CI%20CD.md) · [GitHub Actions](../13%20-%20DevOps%20and%20Delivery/GitHub%20Actions.md) · [Docker](../13%20-%20DevOps%20and%20Delivery/Docker.md) · [Docker Compose](../13%20-%20DevOps%20and%20Delivery/Docker%20Compose.md) · [Kubernetes](../13%20-%20DevOps%20and%20Delivery/Kubernetes.md) · [Terraform](../13%20-%20DevOps%20and%20Delivery/Terraform.md) · [Ansible](../13%20-%20DevOps%20and%20Delivery/Ansible.md) · [Deployment Strategies](../13%20-%20DevOps%20and%20Delivery/Deployment%20Strategies.md) · [Code Review Checklist](../16%20-%20Templates/Code%20Review%20Checklist.md)

---

# MAP 4 — WHAT WRAPS EVERY STATION

**Four concerns that are not stations, because they touch all of them.**

```text
        01 02 03 04 05 06 07 08 09 10 ... 22 23 24
         │  │  │  │  │  │  │  │  │  │       │  │  │
   ┌─────┴──┴──┴──┴──┴──┴──┴──┴──┴──┴───────┴──┴──┴─────┐
   │ IDENTITY        who is allowed to do this?           │
   │   IAM · authentication · authorisation · secrets     │
   │   → in the cloud, IDENTITY IS THE PERIMETER          │
   ├──────────────────────────────────────────────────────┤
   │ OBSERVABILITY   what is happening, and is it OK?     │
   │   metrics (alarm) · logs (investigate) · traces (find│
   │   the slow hop) · SLO + error budget · ONE request id│
   │   flowing through EVERY station above                │
   ├──────────────────────────────────────────────────────┤
   │ SCALE + RELIABILITY   what happens under load, and   │
   │   when a part dies?                                   │
   │   horizontal / vertical · HA · backups · DR ·        │
   │   load testing · timeouts and fallbacks EVERYWHERE   │
   ├──────────────────────────────────────────────────────┤
   │ COST            what does each station cost?          │
   │   compute · EGRESS · requests · log ingestion ·      │
   │   idle capacity · unbounded autoscaling and AI loops │
   └──────────────────────────────────────────────────────┘
```

**IDENTITY** — [IAM](../12%20-%20Cloud%20Architecture/IAM.md) · [Authentication](../09%20-%20Security/Authentication.md) · [Authorization](../09%20-%20Security/Authorization.md) · [Secrets Management](../09%20-%20Security/Secrets%20Management.md) · [Cloud Security](../12%20-%20Cloud%20Architecture/Cloud%20Security.md) · [OWASP Top 10](../09%20-%20Security/OWASP%20Top%2010.md) · [Security Checklist](../16%20-%20Templates/Security%20Checklist.md)

**OBSERVABILITY** — [Monitoring and Alerting](../14%20-%20Scalability%20and%20Reliability/Monitoring%20and%20Alerting.md) · [Cloud Monitoring](../12%20-%20Cloud%20Architecture/Cloud%20Monitoring.md) · [Performance Engineering](../14%20-%20Scalability%20and%20Reliability/Performance%20Engineering.md)

**SCALE + RELIABILITY** — [Scalability Fundamentals](../14%20-%20Scalability%20and%20Reliability/Scalability%20Fundamentals.md) · [Horizontal Scaling](../14%20-%20Scalability%20and%20Reliability/Horizontal%20Scaling.md) · [Vertical Scaling](../14%20-%20Scalability%20and%20Reliability/Vertical%20Scaling.md) · [High Availability](../14%20-%20Scalability%20and%20Reliability/High%20Availability.md) · [Backup Strategy](../14%20-%20Scalability%20and%20Reliability/Backup%20Strategy.md) · [Disaster Recovery](../14%20-%20Scalability%20and%20Reliability/Disaster%20Recovery.md) · [Load Testing](../14%20-%20Scalability%20and%20Reliability/Load%20Testing.md)

**COST** — [Cloud Fundamentals](../12%20-%20Cloud%20Architecture/Cloud%20Fundamentals.md) · [IaaS PaaS SaaS](../12%20-%20Cloud%20Architecture/IaaS%20PaaS%20SaaS.md) · [EC2](../12%20-%20Cloud%20Architecture/EC2.md) · [S3](../12%20-%20Cloud%20Architecture/S3.md)

> [!IMPORTANT]
> **A request id created at station 11 and carried to station 24 is the cheapest architectural decision on this page.** With it, one incident is a single query returning the complete story across every station. Without it, you are correlating timestamps by hand at two in the morning.

---

# MAP 5 — THE FAILURE MAP

**The same journey, read as an adversary would read it. This is the map to reread.**

```text
   STATION              WHAT WAITS THERE
   ─────────────────────────────────────────────────────────────
   01  browser          blocked event loop · no loading state
   02  DNS              TTL of hours · a client that caches forever
   03  TLS              EXPIRED CERTIFICATE  ← startlingly common
   04  request          session token in localStorage + one XSS
   05  network          packet loss looks like slowness
   06  edge             nothing cached · a personalised page cached
   07  VPC              database reachable from the internet
                        · security group open to 0.0.0.0/0
                        · one availability zone
   08  load balancer    a deep health check removing EVERY backend
                        · idle timeout cutting a long request
                        · retries amplifying an overload
   09  proxy            body limit · too few workers, silent queueing
   10  runtime          one blocking call stalls everything
                        · thread pool eaten by a slow dependency
   11  middleware       wrong order → credentials in the logs
                        · no request id, so nothing is traceable
   12  controller       business logic and queries living here
   13  authentication   long-lived JWT → logout does not work
                        · fast password hash → a leak is a takeover
   14  authorisation    ★ CHANGE THE ID IN THE URL, GET SOMEONE
                          ELSE'S DATA — the most common real flaw
   15  validation       trusting a client-sent price · SQL injection
                        · SSRF · a file trusted by its extension
   16  business logic   NO TIMEOUT on a downstream call
                        · a saga with no compensation
   17  cache            ★ CACHE KEY WITHOUT THE TENANT → data leak
                        · caching over a missing index
   18  ORM              ★ N+1: 400 queries for one page
   19  database         missing index · pool exhausted
                        · a long transaction holding locks
   20  the floor        CPU credits · disk IOPS · memory limits —
                        throttling that LOOKS like slow code
   21  queue/worker     ★ NOT IDEMPOTENT → the customer pays twice
                        · no dead letter queue → silent failures
                        · visibility timeout < job duration
   22  response         a stack trace returned · no pagination
   23  egress           large files proxied · full egress paid
   24  render           a 45 ms API inside a 4-second page
   ─────────────────────────────────────────────────────────────
   AND ACROSS ALL OF THEM
      a destructive migration shipped with the code that needs it
      a leaked long-lived credential
      backups in the same account, never restored
      alerts nobody reads
      no rollback that anyone has actually used
```

> [!CAUTION]
> **The five items marked ★ are, in practice, where most real damage comes from — and none of them are exotic.** Missing authorisation, a cache key without a tenant, an N+1 query, a non-idempotent payment handler, and a health check that takes the whole fleet down. They are all cheap to prevent and expensive to discover in production. If you audit one thing after reading this map, audit those five.

---

# STATION INDEX

*Which folder answers which station.*

| Stations | Zone | Folder |
|---|---|---|
| 01, 24 | Their device, rendering | [05 - Frontend Architecture](../05%20-%20Frontend%20Architecture) |
| 02 – 05, 22 | Network, protocols, API shape | [04 - Networking and Internet](../04%20-%20Networking%20and%20Internet) |
| 06, 07, 23 | Edge, cloud, VPC, egress | [12 - Cloud Architecture](../12%20-%20Cloud%20Architecture) |
| 08 | Load balancing, availability | [14 - Scalability and Reliability](../14%20-%20Scalability%20and%20Reliability) |
| 09, 10, 12 | Web servers, runtime, frameworks | [06 - Backend Architecture](../06%20-%20Backend%20Architecture) |
| 10 | Language runtimes and bytecode | [03 - Programming Languages and Runtime](../03%20-%20Programming%20Languages%20and%20Runtime) |
| 11, 12, 15, 16, 18 | Structure of your code | [07 - Backend Design Patterns](../07%20-%20Backend%20Design%20Patterns) |
| 13, 14, 15 | Identity and attacks | [09 - Security](../09%20-%20Security) |
| 17, 18, 19 | Cache, ORM, database | [08 - Databases and Data](../08%20-%20Databases%20and%20Data) |
| 20 | **The floor, under everything** | [02 - Computer Science Fundamentals](../02%20-%20Computer%20Science%20Fundamentals) |
| 21 | Queues, workers, events | [10 - Distributed Systems](../10%20-%20Distributed%20Systems) |
| 21 | The AI branch of the async path | [11 - AI Engineering](../11%20-%20AI%20Engineering) |
| MAP 3 | How the code arrives | [13 - DevOps and Delivery](../13%20-%20DevOps%20and%20Delivery) |
| MAP 4 | Scale, reliability, cost | [14 - Scalability and Reliability](../14%20-%20Scalability%20and%20Reliability) |
| all of it | Real systems, end to end | [15 - Real World System Design](../15%20-%20Real%20World%20System%20Design) |
| — | How to think about all of it | [01 - Foundation](../01%20-%20Foundation) |

---

# THE WHOLE MAP IN SIX SENTENCES

```text
1. A click becomes a network request, and DNS, TCP and TLS all
   happen before a single byte of it reaches you.

2. It should be answered as EARLY as possible — at the edge, or
   in a cache — because everything after that costs real money.

3. Your application asks three questions in this order:
   who are you, may you, and is this valid — and then, finally,
   does the one thing that is actually your product.

4. The database is almost always the real bottleneck, and it is
   standing on a floor of CPU, RAM, disk and kernel that explains
   why every piece of advice above is true.

5. Anything slow does not belong in the request at all — it goes
   on a queue, and the moment it does, it must be idempotent.

6. All of it is wrapped in four things that are not stations:
   identity, observability, reliability, and cost.
```

---

## Personal Notes

*Fill this in yourself.*

- **Stations I could not have named before reading this:**
- **The station where my current system is weakest:**
- **Which of the five ★ failures exist in my system right now:**
- **My own version of MAP 1, drawn from memory:** → [Diagrams](../17%20-%20Personal%20Knowledge%20Base/Diagrams.md)

---

## Workbook Exercise

- [ ] Cover the station names and redraw MAP 1 from memory. Where you stop is what to study next.
- [ ] Take one real request in your own system and walk all 24 stations. Note which ones you do not have, and which you did not know you had.
- [ ] Check the five ★ failures from MAP 5 against your system. Honestly.
- [ ] Find one thing in your system that is answered at station 19 but could be answered at 06 or 17.
- [ ] Find one thing in your request path that belongs at station 21 instead.
- [ ] Pick the station you understand least and read its folder this week.

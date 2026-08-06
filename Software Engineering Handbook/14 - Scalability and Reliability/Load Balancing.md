# Load Balancing

> **In one line —** distributing traffic across several backends; and the health check matters more than the algorithm, because removing a broken server is worth more than choosing cleverly between healthy ones.

| | |
|---|---|
| **Category** | Networking / Traffic Management |
| **Architectural Layer** | Infrastructure |
| **Related notes** | [Horizontal Scaling](Horizontal%20Scaling.md) · [High Availability](High%20Availability.md) · [Scalability Fundamentals](Scalability%20Fundamentals.md) · [Deployment Strategies](../13%20-%20DevOps%20and%20Delivery/Deployment%20Strategies.md) · [DNS](../04%20-%20Networking%20and%20Internet/DNS.md) · [HTTP](../04%20-%20Networking%20and%20Internet/HTTP.md) |

---

## 1. Short Definition

*What is it?*

A load balancer accepts client connections and forwards them to one of several backend servers, **removing unhealthy ones from rotation** and providing a single stable address in front of a changing fleet.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Three servers. Clients need one address.
    ↓
DNS with three records?
    → no health awareness: a dead server still gets a third of the traffic
    → clients cache DNS, so removing a record takes hours to take effect
    → no control over distribution
    ↓
You need something that knows which servers are alive, right now
```

> [!IMPORTANT]
> **The load balancer's most valuable function is not balancing — it is health checking.** Spreading traffic evenly is a modest optimisation; noticing within seconds that one backend is broken and routing around it is what turns a fleet of servers into an available service. Everything else in this note is secondary to getting the health check right.

---

## 3. Architecture Position

```text
                  clients
                     │
              ┌──────▼───────┐
              │     DNS      │  → the load balancer's address
              └──────┬───────┘
              ┌──────▼───────────────────────┐
              │      LOAD BALANCER            │
              │  TLS termination               │
              │  health checks                 │
              │  connection draining           │
              │  routing rules                 │
              └──┬──────────┬──────────┬──────┘
                 ▼          ▼          ▼
              backend    backend    backend
              (healthy) (healthy)  (draining)
```

---

## 4. Layer 4 versus layer 7

```text
LAYER 4 (TCP/UDP)                    LAYER 7 (HTTP)
forwards packets/connections          understands requests
cannot see URLs or headers            can route on path, host, header, cookie
extremely fast, very high throughput  slightly slower, far more capable
preserves the client's TLS end to end  usually terminates TLS
→ NLB, HAProxy in TCP mode            → ALB, NGINX, Envoy, Traefik
```

```text
Use L7 for HTTP services — path routing, header inspection, retries
Use L4 for non-HTTP protocols, extreme throughput, or when TLS must pass through
```

> [!TIP]
> **Almost every web system wants layer 7.** Path-based routing (`/api` to one service, `/` to another), per-route timeouts, header-based canary routing and request retries all require understanding HTTP. Layer 4 earns its place for databases, gRPC streaming at very high volume, game protocols, or where regulatory requirements forbid terminating TLS anywhere but the application.

---

## 5. Algorithms — and why they matter less than you think

```text
ROUND ROBIN            in turn. The default. Usually fine.
LEAST CONNECTIONS      to the backend with fewest active connections
                       → better when request durations VARY a lot
LEAST RESPONSE TIME    to the fastest responder
IP HASH                same client → same backend (a crude form of stickiness)
CONSISTENT HASHING     minimises reshuffling when the backend set changes
                       → important for caches, where key locality matters
WEIGHTED               proportional to declared capacity, or for canaries
RANDOM TWO CHOICES     pick two at random, choose the less loaded
                       → surprisingly close to optimal, and cheap
```

> [!TIP]
> **Round robin is fine until request durations differ widely — then use least connections.** If most requests take 20 ms and a few take 8 seconds, round robin will keep sending new work to a backend already tied up with slow requests. That is the one situation where the algorithm genuinely changes behaviour; the rest of the time it is a rounding error compared with health checks and timeouts.

---

## 6. Health checks: the part to get right

```text
SHALLOW  GET /healthz  → "the process is running"
    ✓ fast, cannot cascade
    ✗ says nothing about whether it can serve

DEEP     GET /ready    → "I can reach my database and dependencies"
    ✓ accurate
    ✗ a shared dependency failing marks EVERY backend unhealthy at once
```

> [!CAUTION]
> **A deep health check that fails when the database is unreachable will remove every backend from rotation simultaneously — turning a degraded database into a total outage, and preventing recovery.** The workable pattern is: the load balancer checks something shallow and local; separate monitoring watches dependencies and pages a human. If you do use a dependency-aware readiness check, ensure the platform will not remove all instances at once — many orchestrators support a minimum-healthy threshold precisely for this.

```text
TUNING
    interval 5-10 s · timeout < interval · 2-3 failures before removal
    → too aggressive: healthy backends flap during a GC pause
    → too lax: broken backends serve errors for a minute
```

---

## 7. Timeouts, retries and draining

```text
CONNECTION DRAINING (deregistration delay)
    stop sending NEW requests, let in-flight ones finish, then remove
    → too short: requests dropped on every deployment
    → set it longer than your slowest normal request

TIMEOUTS
    idle timeout must EXCEED your longest legitimate request
    → a 60 s default silently kills a 90 s report generation

RETRIES
    retry only IDEMPOTENT requests, only on connection failure
    → retrying a POST that timed out may double-charge a customer
```

> [!CAUTION]
> **Retries at the load balancer amplify overload.** When backends are struggling, a retrying load balancer multiplies the load on an already saturated fleet and turns a slowdown into a collapse. Retry on connection-refused, not on timeout; keep the retry budget small; and pair it with circuit breaking so a failing backend stops receiving traffic entirely instead of receiving it twice.

---

## 8. What else lives here

```text
TLS TERMINATION      one place to manage certificates
                     → and one place where traffic is briefly plaintext
HTTP/2 and HTTP/3    modern protocol support without touching the app
COMPRESSION          gzip/brotli at the edge
RATE LIMITING        the right place for it: before your compute
WAF                  request filtering, if you use one
STICKY SESSIONS      cookie-based affinity — see Horizontal Scaling
CANARY WEIGHTING     5% to the new version; see Deployment Strategies
ACCESS LOGS          often your only complete record of what was requested
```

> [!TIP]
> **The load balancer is the correct place for rate limiting, because it is the last point before you start paying for compute.** Rate limiting inside the application still spends a process, a connection and a database lookup on every abusive request. At the edge it costs almost nothing.

---

## 9. Real World Example

- **An ALB in front of ECS or EC2 across two availability zones** — the standard AWS shape; see [AWS Architecture](../12%20-%20Cloud%20Architecture/AWS%20Architecture.md).
- **NGINX or Envoy as an ingress controller** in Kubernetes; see [Kubernetes](../13%20-%20DevOps%20and%20Delivery/Kubernetes.md).
- **A CDN as a global load balancer**, routing users to the nearest healthy region.
- **Blue/green switching** by moving traffic between target groups; see [Deployment Strategies](../13%20-%20DevOps%20and%20Delivery/Deployment%20Strategies.md).
- **Database read balancing** — HAProxy or PgBouncer spreading reads across replicas.
- **Service mesh sidecars** doing per-request load balancing between internal services.

---

## 10. Communication and Dependencies

- **At least two backends** — one backend behind a load balancer is theatre
- **A health endpoint** that is fast, cheap and does not touch dependencies unnecessarily
- **DNS** pointing at the load balancer, with a sensible TTL; see [DNS](../04%20-%20Networking%20and%20Internet/DNS.md)
- **TLS certificates**, ideally automatically renewed
- **Backends in at least two availability zones**, or the balancing achieves nothing during a zone failure
- **Access logs and metrics** — per-target error rates and latency

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use a load balancer for any service with more than one instance, and for any service that must survive an instance failure. Managed load balancers are cheap and remove a whole category of operational work — use one unless you have a specific reason not to.

> [!CAUTION]
> - **Not as a substitute for capacity** — balancing an overloaded fleet just distributes the overload
> - **Not with a single backend**, which gives you an extra hop and no resilience
> - **Not with a deep health check** that turns a dependency failure into total unavailability
> - **Not with retries on non-idempotent requests**
> - **Not with default timeouts** for long-running requests
> - **Not for internal service-to-service calls at high volume** if a mesh or client-side balancing fits better
> - **Not as your only rate limiter** if attacks are a real concern — you also want upstream protection

---

## 12. Advantages and Disadvantages

**Advantages**
- Removes unhealthy backends automatically, in seconds
- One stable address in front of a changing fleet
- Enables horizontal scaling, rolling deployments and canaries
- Centralises TLS, HTTP/2, compression and rate limiting
- Path and host routing lets several services share one entry point
- Cross-zone distribution turns capacity into availability
- Access logs provide a complete request record

**Disadvantages**
- **Another component in the request path**, and another thing to configure wrongly
- Defaults — timeouts, draining, health check intervals — are frequently wrong for your workload
- TLS termination means plaintext exists at that hop
- A managed load balancer is a cost per hour plus per request
- Long-lived connections (WebSockets, gRPC streams) complicate draining and rebalancing
- Can mask backend problems until the whole fleet is degraded
- Sticky sessions undermine most of its benefits

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Added latency** | Typically 1–5 ms for L7; less for L4 |
| **TLS termination** | Offloads CPU from backends — usually a net win |
| **Connection reuse** | Keep-alive to backends avoids repeated handshakes |
| **Throughput** | L4 handles millions of connections; L7 far fewer but still plenty |
| **Warm-up** | Some managed load balancers scale gradually under sudden spikes |
| **Cross-zone balancing** | Evens load, and incurs inter-zone data transfer charges |
| **Uneven long-lived connections** | Once established, connections do not rebalance |

> [!CAUTION]
> **Long-lived connections defeat load balancing after the fact.** With WebSockets or gRPC streams, a backend that joins the fleet after connections are established receives nothing until clients reconnect — so a newly scaled-out instance sits idle while the others stay saturated. The mitigations are a maximum connection age, periodic client reconnection, or a mesh doing per-request balancing.

---

## 14. Security Considerations

> [!CAUTION]
> **After TLS termination, traffic between the load balancer and your backends is only as protected as your network.** In a private subnet that is usually acceptable; for regulated data, re-encrypt to the backend. And be aware the load balancer becomes the enforcement point for a great deal — a permissive listener or a forgotten rule here exposes everything behind it.

- **Only the load balancer is public**; backends live in private subnets; see [VPC](../12%20-%20Cloud%20Architecture/VPC.md)
- **Security groups reference the load balancer's group**, so backends accept nothing else
- **TLS 1.2 minimum**, modern cipher policy, HSTS, and automatic certificate renewal
- **Rate limiting and connection limits** at the edge
- **Trust `X-Forwarded-For` only from your own load balancer** — clients can forge it, and IP-based rules built on a forged header are worse than none
- **Access logs to durable storage**, since they may be your only forensic record
- **Health endpoints must not leak information** — no versions, no dependency detail, no stack traces
- **Slowloris and connection exhaustion** are handled at this layer, not in the application

---

## 15. Mental Model

> [!NOTE]
> **A load balancer is the host at a restaurant, not a waiter.**
>
> Guests arrive at one door and are shown to a table — the host knows which tables are free, which server is overwhelmed, and crucially which table has a broken chair and should be skipped entirely. Spotting the broken chair is the job that matters; whether guests are seated strictly in rotation is a detail. And if every table is full, a better seating policy does not create capacity.

---

## 16. Mini Architecture Diagram

```text
                        internet
                            │
                    DNS → load balancer
                            │
   ┌────────────── LOAD BALANCER (AZ-a + AZ-b) ────────────────┐
   │  TLS 1.2+ termination · HTTP/2 · gzip                      │
   │  rate limiting  ← cheapest possible place                  │
   │  routing:  /api/*  → api target group                      │
   │            /*      → web target group                      │
   │  idle timeout > slowest legitimate request                  │
   │  deregistration delay > slowest normal request              │
   │  health check: GET /healthz  (SHALLOW, local only)          │
   │  retries: connection failures only, small budget            │
   └───┬──────────────────┬──────────────────┬─────────────────┘
       ▼                  ▼                  ▼
  ┌─────────┐        ┌─────────┐       ┌──────────┐
  │ AZ-a    │        │ AZ-b    │       │  AZ-a    │
  │ healthy │        │ healthy │       │ DRAINING │  ← in-flight
  └────┬────┘        └────┬────┘       └──────────┘    requests finish
       └────────┬─────────┘
                ▼
        shared database, cache        ← NOT part of the health check
                                        (monitored separately)
```

---

## 17. Complete Request Flow

```text
Client resolves the DNS name → the load balancer's address
    ↓
TLS handshake terminated at the load balancer
    ↓
Path is /api/orders → routed to the api target group
    ↓
Least-connections picks the backend with fewest active requests
    ↓
Forwarded over a reused keep-alive connection (no new handshake)
    ↓
X-Forwarded-For added; the application trusts it only from this source
    ↓
Response returns; access log records target, latency and status
    ↓
─────────────── a backend degrades ───────────────
Backend 2 starts returning 500s but keeps passing /healthz
    ↓
The load balancer does not notice — the health check is shallow
    ↓
BUT per-target error rate metrics alarm, and a human is paged
    ↓
LESSON: health checks catch dead backends; monitoring catches sick ones
    ↓
─────────────── a backend dies ───────────────
Backend 2 stops responding entirely
    ↓
Two consecutive health check failures over 10 s → removed from rotation
    ↓
Traffic redistributed to the remaining backends
    ↓
The scaling group replaces it; it rejoins once healthy
    ↓
─────────────── deployment ───────────────
New version registered; old backends set to draining
    ↓
No new requests to them; in-flight requests complete within the delay
    ↓
Zero dropped requests — because the delay exceeded the slowest request
    ↓
─────────────── overload ───────────────
Traffic triples; every backend is saturated
    ↓
Health checks still pass; latency climbs
    ↓
Retries would make this worse, so the retry budget is small
    ↓
Rate limiting at the edge sheds excess load, protecting the fleet
    ↓
The real fix is capacity — the load balancer can only distribute it
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Get the health check, timeouts and draining right before worrying about the algorithm; keep the check shallow so a dependency failure cannot remove every backend at once, and put rate limiting here rather than in your application.

---

## 19. Common Mistakes

- **A deep health check** that removes the whole fleet when one dependency fails
- **Health checks so aggressive** that healthy backends flap during GC pauses
- **Default idle timeout** silently killing long-running legitimate requests
- **Deregistration delay shorter than the slowest request**, dropping requests on every deployment
- **Retrying non-idempotent requests**, causing duplicate side effects
- **Retries during overload**, amplifying a slowdown into an outage
- **All backends in one availability zone**
- **A single backend** behind a load balancer, providing no resilience
- **Trusting `X-Forwarded-For`** from anywhere
- **Sticky sessions by default**, undermining failover and even distribution
- **Health endpoints that leak** version and dependency information
- **Expecting balancing to fix an overloaded fleet**
- **Ignoring long-lived connection imbalance** after scaling out

---

## 20. Open Source Technologies

- **NGINX** — the workhorse: reverse proxy, L7 balancing, TLS, caching
- **HAProxy** — exceptional at L4 and L7, with the best observability of the classic tools
- **Envoy** — modern, dynamic configuration, retries, circuit breaking, outlier detection
- **Traefik** — automatic service discovery and certificates; strong in containerised setups
- **Caddy** — automatic HTTPS with almost no configuration
- **Keepalived / VRRP** — a floating IP for on-premises load balancer redundancy
- **PgBouncer**, **ProxySQL** — connection pooling and balancing for databases
- **Istio**, **Linkerd** — per-request balancing between services in a mesh
- **k6**, **wrk** — verify that your configuration behaves under load

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Read your health check endpoint. Does it touch the database? Decide whether it should.
- [ ] Compare your deregistration delay with your slowest normal request.
- [ ] Compare your idle timeout with your longest legitimate request.
- [ ] Check whether retries are enabled and whether they could duplicate a side effect.
- [ ] Kill a backend and time how long until traffic stops reaching it.
- [ ] Confirm your backends span at least two availability zones.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
DNS → load balancer (TLS, routing, rate limit, health checks) → healthy backends across AZs
```

## 2. Request Flow

```text
Input       a client connection to one stable address
    ↓
Processing  TLS terminated, route matched, a healthy backend selected
    ↓
Output      a response — and a fleet where broken backends stop receiving traffic
```

## 3. Real-World Usage

Every load-balanced production system converges on the same short list of things that actually matter: a shallow health check, a draining delay longer than the slowest request, an idle timeout that does not truncate legitimate work, and rate limiting at the edge. Teams argue about algorithms; incidents are caused by the four settings above.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A component that distributes traffic across healthy backends behind one address |
| **Why does it exist?** | Because DNS cannot tell which servers are alive, and clients cache it anyway |
| **Where does it belong?** | In front of every tier that has more than one instance |
| **When should I use it?** | Whenever more than one backend exists, or one failing must not matter |

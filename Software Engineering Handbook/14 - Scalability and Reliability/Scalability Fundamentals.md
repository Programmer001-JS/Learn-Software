# Scalability Fundamentals

> **In one line —** the ability to handle more work by adding resources; and the reason it is hard is that one un-parallelisable part of your system sets the ceiling for everything else.

| | |
|---|---|
| **Category** | Overview note *(hub for this sub-section)* |
| **Architectural Layer** | Cross-cutting |
| **Sub-topics** | [Horizontal Scaling](Horizontal%20Scaling.md) · [Vertical Scaling](Vertical%20Scaling.md) · [Load Balancing](Load%20Balancing.md) · [Performance Engineering](Performance%20Engineering.md) · [Load Testing](Load%20Testing.md) |
| **Related notes** | [High Availability](High%20Availability.md) · [Cache](../08%20-%20Databases%20and%20Data/Cache.md) · [Microservices](../10%20-%20Distributed%20Systems/Microservices.md) · [Cloud Fundamentals](../12%20-%20Cloud%20Architecture/Cloud%20Fundamentals.md) |

---

## 1. Short Definition

*What is it?*

Scalability is the property that **adding resources increases capacity**. A system that doubles its throughput when you double its servers scales well. A system that gains 20% scales badly — and the difference is almost never the servers.

---

## 2. Problem

*What engineering problem does it solve?*

```text
100 users        → everything is fine
1,000 users      → a little slow
10,000 users     → the database is at 100% CPU
100,000 users    → adding servers changes nothing
    ↓
Because the bottleneck was never the servers
```

> [!IMPORTANT]
> **Scalability is not about being fast; it is about the shape of the curve.** A system that serves one request in 50 ms and ten thousand concurrent requests in 50 ms scales. A system that serves one request in 5 ms and ten thousand in 8 seconds does not — even though it was faster to begin with. Optimising the single-request case and optimising the curve are different activities, and teams routinely do the first while believing they are doing the second.

---

## 3. Architecture Position

```text
   users
     │
   CDN / edge            ← scales trivially; use it first
     │
   load balancer         ← scales horizontally
     │
   ┌───┴────┬────────┐
   app    app      app   ← STATELESS: scales horizontally, easily
   └───┬────┴────────┘
       │
     cache                ← absorbs read load
       │
   ┌───┴──────────┐
   database        ← THE BOTTLENECK, almost always
   (writes are the hard part)
```

> [!TIP]
> **Scalability work moves down this stack in order, and the cost rises at every step.** Caching and a CDN are configuration. Stateless application servers are a design property you can achieve. Scaling the database's *writes* is genuinely difficult and often architectural. Anyone doing this in a different order is spending the expensive effort first.

---

## 4. Amdahl's law, in plain terms

```text
If 5% of your work cannot be parallelised,
the maximum speed-up is 20×, with INFINITE resources.
```

```text
   serial portion    maximum speed-up
        1%                100×
        5%                 20×
       10%                 10×
       25%                  4×
```

> [!IMPORTANT]
> **Find the serial part, because it is your ceiling and no amount of hardware raises it.** In real systems the serial part is usually a single database primary accepting all writes, a global lock, a queue with one consumer, or a third-party API with a rate limit. This is why "just add servers" stops working: the added servers all wait on the same thing. The productive question is never "how many servers?" but "what is everything waiting for?"

---

## 5. The two directions

```text
VERTICAL (scale up)              HORIZONTAL (scale out)
a bigger machine                  more machines
no code changes                   requires statelessness
simple, instant                   near-unlimited headroom
HARD CEILING                      needs a load balancer
one machine = one failure point   survives a machine loss
```

```text
The honest sequence
    1. scale up first — it is cheap and buys years
    2. scale out when the ceiling or availability demands it
    3. shard only when a single writer is genuinely the limit
```

> [!TIP]
> **Vertical scaling is underrated and horizontal scaling is over-prescribed.** A modern single machine can have 128 cores and terabytes of RAM; most applications that "need to scale horizontally" would run comfortably on one large server with an index added. Scale up until it stops working or until availability requires redundancy — see [Vertical Scaling](Vertical%20Scaling.md) and [Horizontal Scaling](Horizontal%20Scaling.md).

---

## 6. State is what makes scaling hard

```text
STATELESS                            STATEFUL
any server can serve any request     this server holds something unique
add and remove freely                moving it means moving data
a crash loses nothing                a crash loses something
    ↓                                    ↓
scales by adding copies              scales only by partitioning
```

```text
WHERE STATE SHOULD LIVE
    session data      → a shared cache (Redis), or a signed token
    uploaded files    → object storage (S3)
    the truth         → a database
    inside the app    → NOTHING durable
```

> [!IMPORTANT]
> **"Make the application layer stateless" is the highest-leverage sentence in this note.** In-process sessions force sticky routing; local file uploads mean one server holds data others cannot see; in-memory counters and caches diverge between instances. Push all of it outward and the application tier becomes something you can add to and remove from freely — which is what horizontal scaling actually requires.

---

## 7. Reads and writes are different problems

```text
READS                                WRITES
cache them                            cannot be cached
add read replicas                     usually ONE primary
CDN the static ones                   ordering and consistency matter
→ scale almost arbitrarily            → the real ceiling
```

```text
SCALING WRITES, in increasing order of pain
    batch them
    make them asynchronous (queue + worker)
    partition by tenant or entity  ← sharding
    relax consistency where the business genuinely allows it
```

> [!CAUTION]
> **Sharding is the last resort, not a milestone.** Once data is partitioned, cross-shard joins, transactions and unique constraints become application problems, rebalancing becomes an operation, and every query needs a shard key. Do it when a single primary on the largest available instance genuinely cannot keep up — and expect the migration to be the largest project of the year.

---

## 8. Queues: the pressure valve

```text
SYNCHRONOUS                          ASYNCHRONOUS
user waits for everything             user waits for the essential part
    upload → transcode → email        upload → enqueue → respond
    → 40 seconds                      → 200 ms; work happens behind
peak load = peak capacity needed      queue absorbs the spike
one slow dependency = slow request    workers scale independently
```

> [!TIP]
> **Moving work off the request path is often a bigger win than any optimisation.** It converts a capacity problem into a latency-of-completion problem, which is usually much cheaper. The requirement is that queue depth and message age are monitored — an unbounded queue is not resilience, it is a delayed outage; see [Message Queues](../10%20-%20Distributed%20Systems/Message%20Queues.md).

---

## 9. Real World Example

- **A read-heavy content site** — a CDN plus caching handles almost everything; the database barely notices.
- **A ticketing system** — a genuinely serial constraint (one seat, one buyer) that no amount of hardware removes.
- **Uber** — geographic partitioning as the natural shard key; see [Uber Architecture](../15%20-%20Real%20World%20System%20Design/Uber%20Architecture.md).
- **Netflix** — the heavy bytes moved to the edge, so the core serves only metadata; see [Netflix Architecture](../15%20-%20Real%20World%20System%20Design/Netflix%20Architecture.md).
- **Video processing** — a queue with autoscaled workers, where latency to completion is elastic; see [Video Processing Platform](../15%20-%20Real%20World%20System%20Design/Video%20Processing%20Platform.md).
- **Multi-tenant SaaS** — tenant id is the shard key, and the architecture is simpler because of it.

---

## 10. Communication and Dependencies

- **[Load Balancing](Load%20Balancing.md)** — required by any horizontal tier
- **[Cache](../08%20-%20Databases%20and%20Data/Cache.md)** — the cheapest capacity you will ever add
- **Shared state stores** — a cache and object storage, so the application can be stateless
- **[Monitoring and Alerting](Monitoring%20and%20Alerting.md)** — you cannot scale what you cannot measure
- **[Load Testing](Load%20Testing.md)** — the only way to know where the bottleneck actually is
- **Connection pooling** — the database's connection limit is a ceiling people meet unexpectedly

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Design for the traffic you have plus one order of magnitude. Statelessness, caching and asynchronous work are cheap to adopt early and expensive to retrofit — so do those. Sharding, event sourcing and elaborate distribution are not.

> [!CAUTION]
> - **Do not scale before measuring** — the bottleneck is rarely where the intuition says
> - **Do not distribute a system that fits on one machine** — you trade a solvable problem for a harder one
> - **Do not build for a million users** when you have a thousand; the design that works at a million is usually wrong at a thousand
> - **Do not scale to fix an inefficient query** — an index costs nothing and often removes the need entirely
> - **Do not add servers to a system limited by its database primary**; you are adding queue length
> - **Do not confuse scalability with availability** — related, and not the same; see [High Availability](High%20Availability.md)

---

## 12. Advantages and Disadvantages

**Advantages of designing for scale**
- Capacity becomes a purchasing decision rather than a rewrite
- Stateless tiers also give you easier deployment and self-healing
- Caching and asynchronous work reduce cost as well as latency
- Traffic spikes stop being incidents
- Most of the same properties improve reliability

**Disadvantages**
- **Complexity, which has an ongoing cost in every debugging session**
- Distributed systems introduce partial failure, retries and idempotency requirements
- Caching introduces staleness and invalidation bugs
- Asynchronous flows are harder to reason about and to test
- Sharding removes joins, transactions and easy analytics
- Premature scaling work is engineering time not spent on the product

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Latency vs throughput** | Different goals; optimising one can worsen the other |
| **Percentiles** | p99 is what users feel; averages conceal the problem |
| **Queueing** | Beyond roughly 70–80% utilisation, latency rises non-linearly |
| **Connection limits** | Often the first hard ceiling met, and rarely anticipated |
| **Cache hit ratio** | The difference between a database that copes and one that does not |
| **Cross-AZ and cross-region calls** | Every network hop is added latency and added failure surface |

> [!CAUTION]
> **A system at 85% utilisation is not "efficient", it is on the edge of a cliff.** Queueing theory says latency approaches infinity as utilisation approaches 100%, and the curve turns sharply well before that. This is why systems appear fine and then collapse suddenly rather than degrading smoothly. Target 50–70% at peak and treat the headroom as the cost of predictable latency.

---

## 14. Security Considerations

> [!CAUTION]
> **Scalability and abuse resistance are the same problem viewed differently.** A system that scales by adding capacity in response to load will scale in response to an attack too — and bill you for it. Rate limiting, quotas and per-tenant fairness must be part of the design, not a reaction. Autoscaling without an upper bound turns a denial-of-service attempt into a financially successful one.

- **Rate limit per client and per tenant**, at the edge
- **Upper bounds on autoscaling**, so cost cannot run away
- **A noisy tenant must not starve others** — fair queueing and per-tenant quotas
- **Cache keys must include the authorisation context**, or a cache becomes a cross-user data leak
- **Retries need backoff and jitter**, or your own clients will amplify an incident into an outage
- **A queue is a resource too** — an attacker who can enqueue can exhaust workers

---

## 15. Mental Model

> [!NOTE]
> **A system scales like a motorway, and the bottleneck is always the toll booth.**
>
> Adding lanes helps right up to the point where every car must pass one booth — after which more lanes just make a wider queue. Widening to eight lanes when the booth is the constraint is expensive and changes nothing. The real options are more booths (horizontal), a faster booth (vertical), letting some cars through without stopping (caching), or sending them by a different route entirely (queues and asynchronous work).

---

## 16. Mini Architecture Diagram

```text
                        users
                          │
                  ┌───────▼────────┐
                  │  CDN / edge    │  static + cached: scales for free
                  └───────┬────────┘
                          │  rate limited here
                  ┌───────▼────────┐
                  │ load balancer  │
                  └───────┬────────┘
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
      ┌───────┐       ┌───────┐       ┌───────┐
      │ app   │       │ app   │       │ app   │   STATELESS
      │ 60%   │       │ 60%   │       │ 60%   │   (headroom, not 90%)
      └───┬───┘       └───┬───┘       └───┬───┘
          └───────┬───────┴───────┬───────┘
                  ▼               ▼
            ┌──────────┐   ┌─────────────┐
            │  cache   │   │   QUEUE     │──► workers (autoscaled,
            │  (Redis) │   │ depth+age   │     capped)
            └────┬─────┘   └─────────────┘
                 │  absorbs reads
     ┌───────────▼────────────────────┐
     │  DATABASE                       │
     │   primary  ← ALL WRITES         │  ◄── the serial part:
     │      │                          │      your actual ceiling
     │   replicas ← reads              │
     └────────────────────────────────┘
```

---

## 17. Complete Request Flow

```text
Traffic grows 10× over six months
    ↓
STAGE 1 — measure
    p99 latency has tripled; the database is at 90% CPU
    Adding app servers changed nothing → the bottleneck is below them
    ↓
STAGE 2 — the cheapest fixes first
    One missing index removes 40% of database CPU
    A CDN in front of static assets removes 60% of requests entirely
    ↓
STAGE 3 — cache
    Read-through cache on the hottest queries: 85% hit ratio
    Database CPU now 35%
    ↓
STAGE 4 — statelessness
    Sessions moved from process memory to Redis
    Uploads moved from local disk to S3
    → now any app server can serve any request; sticky sessions removed
    ↓
STAGE 5 — asynchronous work
    Transcoding and email moved to a queue
    Request latency drops from 3 s to 180 ms
    Workers autoscale on queue depth, with a cap
    ↓
STAGE 6 — read replicas
    Reporting and analytics moved off the primary
    ↓
STAGE 7 — vertical, first
    Primary moved to a larger instance: another 4× headroom for one afternoon's work
    ↓
STAGE 8 — only now, sharding
    Writes genuinely exceed one primary
    Partition by tenant id — a year-long project, undertaken deliberately
    ↓
─────────────── what would have gone wrong ───────────────
Starting at stage 8 would have taken a year and solved nothing,
because the actual constraint was one missing index and no cache.
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Find the serial bottleneck before adding anything — then take the cheap wins in order: index, CDN, cache, statelessness, queues, replicas, bigger machine, and only then shard.

---

## 19. Common Mistakes

- **Adding servers** to a system bottlenecked on its database
- **Optimising single-request latency** and calling it scalability work
- **Never measuring**, so the bottleneck is guessed and the guess is wrong
- **Skipping vertical scaling**, which was cheap and would have bought years
- **Keeping state in the application** — sessions, uploads, in-memory counters
- **Sharding early**, losing joins and transactions for nothing
- **Running at 85% utilisation** and being surprised by sudden collapse
- **Watching averages** rather than p99
- **Unbounded queues** treated as resilience
- **Autoscaling with no upper limit**, turning load into an unbounded bill
- **Caching without considering authorisation**, leaking data between users
- **Retries without backoff**, amplifying a small failure into an outage
- **Building for a million users** while serving a thousand

---

## 20. Open Source Technologies

- **NGINX**, **HAProxy**, **Envoy** — load balancing and edge rate limiting
- **Redis**, **Memcached** — shared state and caching
- **Kafka**, **RabbitMQ**, **NATS** — queues and asynchronous work
- **PostgreSQL** with **PgBouncer** — and read replicas before anything exotic
- **Citus**, **Vitess** — sharding for PostgreSQL and MySQL when it is genuinely time
- **k6**, **Locust**, **Vegeta**, **wrk** — load testing; see [Load Testing](Load%20Testing.md)
- **Prometheus**, **Grafana**, **OpenTelemetry** — measure before you change
- **Varnish** — HTTP caching in front of an application

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Name your system's serial bottleneck. If you cannot, that is the first task.
- [ ] Find one thing your application stores locally that should be in a shared store.
- [ ] Check your peak utilisation. If it is above 80%, you have less headroom than you think.
- [ ] Identify one synchronous piece of work that could move to a queue.
- [ ] Compare your p50 and p99. If the gap is large, you have a queueing problem, not a speed problem.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
CDN → load balancer → stateless app tier → cache → database primary (the ceiling)
                                      └→ queue → autoscaled workers
```

## 2. Request Flow

```text
Input       more load than the system currently handles
    ↓
Processing  find what everything waits on; remove or bypass it, cheapest first
    ↓
Output      capacity that grows with resources instead of flattening
```

## 3. Real-World Usage

The systems that scale well in practice are unremarkable: static content at the edge, a stateless application tier, aggressive caching, asynchronous work behind queues, and a relational database kept comfortable with indexes and replicas. The exotic techniques appear only where a genuine serial constraint forced them — and usually much later than the teams expected.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The property that adding resources adds capacity |
| **Why does it exist as a discipline?** | Because one un-parallelisable component caps the whole system |
| **Where does it belong?** | Everywhere, but the work concentrates at the data layer |
| **When should I use it?** | Design for one order of magnitude beyond today; measure before every change |

# Horizontal Scaling

> **In one line —** adding more machines rather than bigger ones; nearly unlimited headroom, bought with the requirement that your application hold no state.

| | |
|---|---|
| **Category** | Scaling Strategy |
| **Architectural Layer** | Infrastructure |
| **Related notes** | [Scalability Fundamentals](Scalability%20Fundamentals.md) · [Vertical Scaling](Vertical%20Scaling.md) · [Load Balancing](Load%20Balancing.md) · [High Availability](High%20Availability.md) · [EC2](../12%20-%20Cloud%20Architecture/EC2.md) · [Kubernetes](../13%20-%20DevOps%20and%20Delivery/Kubernetes.md) |

---

## 1. Short Definition

*What is it?*

Horizontal scaling — scaling *out* — means handling more load by running **more instances of the same thing** behind a load balancer, instead of making one instance larger.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Vertical scaling runs out
    ↓
The largest instance available is still not enough
And one machine means one failure takes everything down
And you pay for peak capacity 24 hours a day
    ↓
You need capacity that can grow past one machine — and survive losing one
```

> [!IMPORTANT]
> **Horizontal scaling buys two different things at once, and the second is usually more valuable.** The headroom is nearly unlimited, yes — but the reason most systems run more than one instance is not throughput, it is that **a single instance is a single point of failure.** Even a system that comfortably fits on one machine should run two.

---

## 3. Architecture Position

```text
                    load balancer
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   ┌─────────┐       ┌─────────┐       ┌─────────┐
   │instance │       │instance │       │instance │   identical,
   │  AZ-a   │       │  AZ-b   │       │  AZ-a   │   interchangeable,
   └────┬────┘       └────┬────┘       └────┬────┘   disposable
        └─────────┬───────┴────────┬─────────┘
                  ▼                ▼
            shared cache      shared database
            (sessions)        (the truth)
                              object storage (files)
```

Everything unique lives **below** the instance line. Everything above it is a copy.

---

## 4. The precondition: statelessness

```text
STATE THAT BREAKS HORIZONTAL SCALING
    sessions in process memory    → user logged out at random
    uploaded files on local disk  → 404 on two out of three servers
    in-memory caches              → inconsistent answers per instance
    a scheduled job in every copy → the email sends three times
    local write-ahead logs        → data lost when the instance is replaced
```

```text
WHERE IT GOES INSTEAD
    sessions   → Redis, or a signed stateless token
    files      → object storage (S3)
    cache      → a shared cache, with per-instance caching as a bonus only
    scheduling → a single leader, a distributed lock, or a scheduler service
```

> [!CAUTION]
> **Scheduled jobs are the state problem people forget.** Application code that runs a nightly report on a timer works perfectly on one instance and sends three reports on three instances. The fix is a leader election, a database lock, or moving the trigger out to a scheduler that enqueues one message — but it has to be a deliberate decision, because nothing will warn you.

---

## 5. Sticky sessions: the workaround that costs you

```text
STICKY SESSIONS (session affinity)
    the load balancer sends each user back to the same instance
    → lets a stateful app "work" behind a load balancer
```

```text
WHAT IT BREAKS
    an instance dies    → those users lose their sessions
    load is uneven      → one instance gets the heavy users
    scaling out         → new instances receive no existing traffic
    deployments         → draining takes as long as the longest session
```

> [!TIP]
> **Sticky sessions are a legitimate short-term bridge and a bad long-term design.** They let you scale out today without touching the application — and they preserve every problem statelessness was meant to solve. If you turn them on, treat it as debt with a date attached: move sessions to Redis or a signed token, then turn affinity off and watch the deployment and failover behaviour improve.

---

## 6. Autoscaling: get the signal right

```text
SCALE ON WHAT ACTUALLY CORRELATES WITH LOAD
    web tier        → requests per instance, or p95 latency
    worker tier     → QUEUE DEPTH or message age   ← the best signal there is
    CPU-bound work  → CPU utilisation
    → CPU is the default and often the WRONG metric for I/O-bound services
```

```text
GET THE TIMING RIGHT
    scale OUT fast, scale IN slowly
    a cooldown, or you oscillate (thrash)
    warm-up time counts: if an instance takes 5 minutes to be ready,
    your response to a spike arrives after the spike
```

> [!IMPORTANT]
> **Autoscaling reacts, so it is always late.** By the time the metric crosses the threshold, users are already queueing; and then you wait for provisioning, boot and warm-up. This is why the two things that make autoscaling work are **fast startup** — bake images, use containers, keep health checks quick — and **headroom**, so you are scaling from comfortable rather than from saturated. Predictive or scheduled scaling helps where the pattern is known.

---

## 7. What does not scale by adding copies

```text
SCALES WELL by adding instances
    stateless web and API servers
    queue workers
    read replicas (for reads)
    stateless transformations

DOES NOT
    a single database primary accepting writes    ← the usual real ceiling
    anything holding a global lock
    a third-party API with a rate limit
    a filesystem that must be shared and consistent
```

> [!CAUTION]
> **Adding application instances to a system bottlenecked on its database primary makes things worse, not better.** More instances open more connections, each contending for the same locks and the same CPU — so latency rises while throughput does not. The measurable symptom is that adding capacity does nothing or degrades p99, and the correct response is to look one layer down. See [Scalability Fundamentals](Scalability%20Fundamentals.md).

---

## 8. Distribute across failure domains

```text
3 instances, ALL in AZ-a       → an AZ outage takes 100% of capacity
2 instances, one per AZ        → an AZ outage takes 50%
4 instances, two per AZ        → an AZ outage leaves 50%, sized for it
```

```text
The rule: size the fleet so that losing a whole failure domain
still leaves enough capacity — not just enough instances.
```

> [!TIP]
> **Spreading instances across availability zones is what converts "more capacity" into "more availability".** Three instances in one zone give throughput and no resilience. The cost is cross-zone data transfer charges, which are real but small next to an outage; see [High Availability](High%20Availability.md).

---

## 9. Real World Example

- **Any web or API tier behind a load balancer** — the standard shape, and the easy case.
- **Queue workers autoscaled on queue depth** — the cleanest example of the pattern working well.
- **Kubernetes Deployments with an HPA** — horizontal scaling as a first-class API object; see [Kubernetes](../13%20-%20DevOps%20and%20Delivery/Kubernetes.md).
- **CI runner fleets** on spot capacity, where an interruption costs one retried job.
- **Read replicas** for reporting and search, scaling reads without touching writes.
- **Stateless inference services**, scaled on request rate; see [Model Serving](../11%20-%20AI%20Engineering/Model%20Serving.md).

---

## 10. Communication and Dependencies

- **A load balancer** with health checks; see [Load Balancing](Load%20Balancing.md)
- **A shared session and cache store** — Redis, or stateless tokens
- **Object storage** for anything file-shaped
- **Connection pooling** — N instances × pool size can exhaust the database
- **A metric that reflects load**, and a scaling policy built on it
- **Fast, immutable startup** — a baked image or a container
- **Idempotent handlers**, because retries and duplicate delivery become normal

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Scale horizontally when you need more capacity than one machine provides, when you need to survive losing a machine, or when load varies enough that paying for peak all day is wasteful. Run at least two instances across two zones even for small systems — the availability alone justifies it.

> [!CAUTION]
> - **Not before removing state** — sticky sessions will hide the problem, not solve it
> - **Not for the database's write path** — that needs replication or sharding, not copies
> - **Not when vertical scaling is simpler and sufficient** — see [Vertical Scaling](Vertical%20Scaling.md)
> - **Not without connection pooling** — you will hit the database's connection ceiling
> - **Not with slow startup** — autoscaling arrives after the spike
> - **Not without an upper bound**, or an attack becomes an unbounded bill

---

## 12. Advantages and Disadvantages

**Advantages**
- Nearly unlimited headroom
- Survives the loss of an instance, a rack or a whole zone
- Elastic: capacity follows demand, and cost follows capacity
- Enables rolling deployments and canary releases
- Commodity instances are cheaper per unit than the largest ones
- Spot and preemptible capacity becomes usable

**Disadvantages**
- **Requires a stateless application**, which is a design constraint, not a setting
- A load balancer, health checks and service discovery to operate
- Distributed problems arrive: retries, duplicates, partial failure, clock skew
- Debugging spans many instances — you need aggregated logs and traces
- Per-instance caches are less effective than one large one
- Connection count against shared dependencies multiplies
- Cross-zone traffic costs money

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Throughput** | Scales close to linearly, until a shared dependency saturates |
| **Latency** | Unchanged per request; one extra load balancer hop |
| **Cache hit ratio** | **Falls** as you add instances, if caching is in-process |
| **Connections** | Instances × pool size — a common surprise ceiling |
| **Warm-up** | Cold caches and JIT warm-up make new instances temporarily slow |
| **Scale-out latency** | Seconds for containers, minutes for instances |

> [!TIP]
> **Watch the in-process cache hit ratio as you scale out.** Ten instances each caching independently means each sees a tenth of the traffic and a tenth of the repeats. This is a real and often unnoticed regression when scaling out — the fix is a shared cache tier, with the local cache kept only as a short-lived first layer.

---

## 14. Security Considerations

> [!CAUTION]
> **Unbounded autoscaling turns a denial-of-service attempt into a financial attack.** Without a maximum instance count and rate limiting at the edge, an attacker cannot take your service down but can spend your budget. Set an upper bound on every scaling policy and rate limit before the load balancer.

- **A maximum on every autoscaling policy**
- **Rate limiting and abuse controls at the edge**, ahead of the fleet
- **Every instance is an attack surface** — identical, immutable images make patching tractable
- **No credentials on instances** — use attached roles; see [IAM](../12%20-%20Cloud%20Architecture/IAM.md)
- **Instances in private subnets**, reachable only through the load balancer; see [VPC](../12%20-%20Cloud%20Architecture/VPC.md)
- **Aggregated, centralised logs** — an instance's local logs vanish when it is replaced, including evidence
- **Session storage becomes a critical dependency** — secure and back the shared store accordingly

---

## 15. Mental Model

> [!NOTE]
> **Horizontal scaling is opening more checkout lanes; vertical scaling is training one cashier to be faster.**
>
> More lanes serve more customers and keep the shop open when one cashier is ill — but only if any lane can serve any customer. The moment a customer's shopping is stored behind one particular till (state), extra lanes stop helping and you need to send that person back to the same one every time (sticky sessions). And none of it matters if there is a single price-check desk everybody must visit.

---

## 16. Mini Architecture Diagram

```text
                    internet
                        │
                rate limiting at the edge
                        │
              ┌─────────▼──────────┐
              │   load balancer    │  health checks, connection draining
              └─────────┬──────────┘
       ┌────────────────┼────────────────┐
       │  AZ-a          │        AZ-b    │
   ┌───▼────┐      ┌────▼───┐      ┌────▼───┐
   │instance│      │instance│      │instance│
   │ 60%    │      │ 60%    │      │ 60%    │   headroom, not 90%
   └───┬────┘      └────┬───┘      └────┬───┘
       └──────┬─────────┴───────┬────────┘
              ▼                 ▼
     ┌────────────────┐  ┌──────────────────┐
     │  Redis         │  │  connection pool │
     │  sessions +    │  │  → DB primary    │  ◄── the shared ceiling
     │  shared cache  │  │  → replicas      │
     └────────────────┘  └──────────────────┘
              │
     ┌────────▼────────┐
     │ object storage  │  uploads — never local disk
     └─────────────────┘

   AUTOSCALING: metric = requests/instance (web) or queue depth (workers)
                min 2, max BOUNDED, scale out fast, in slowly
```

---

## 17. Complete Request Flow

```text
Traffic doubles over an hour
    ↓
Requests per instance crosses the target; the scaling policy triggers
    ↓
Two instances launched, one per AZ, from a pre-baked image (45 s to ready)
    ↓
Health checks pass → the load balancer adds them to rotation
    ↓
New instances have cold caches, so they are briefly slower
    ↓
They read sessions from Redis, so any user can land on any instance
    ↓
Capacity absorbed; p99 returns to normal
    ↓
─────────────── an instance fails ───────────────
One instance stops responding to health checks
    ↓
Removed from rotation within seconds; in-flight requests drained
    ↓
The scaling group terminates and replaces it
    ↓
No user was logged out, because no session lived on that instance
    ↓
─────────────── an AZ fails ───────────────
AZ-a is lost: half the fleet disappears at once
    ↓
The load balancer routes everything to AZ-b
    ↓
The fleet was sized so that half of it can serve peak — degraded, not down
    ↓
Autoscaling adds instances in the surviving zone
    ↓
─────────────── scaling stops helping ───────────────
Traffic keeps rising; instances are added; p99 gets WORSE
    ↓
Database CPU at 95%, connection count at the limit
    ↓
Adding instances was adding contention
    ↓
The real fix is one layer down: caching, replicas, a bigger primary
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Remove state before you scale out, run at least two instances across two zones, autoscale on a metric that reflects real load with a bounded maximum, and remember that adding instances cannot fix a bottleneck below them.

---

## 19. Common Mistakes

- **Scaling out with state still in the application**, then papering over it with sticky sessions
- **Sticky sessions left permanently**, keeping every failover and deployment problem
- **Scheduled jobs running on every instance**, so everything happens N times
- **Autoscaling on CPU** for an I/O-bound service
- **No cooldown**, causing constant scale-out and scale-in thrash
- **Slow instance startup**, so scaling responds after the spike has passed
- **No maximum instance count**, turning a traffic attack into a billing one
- **All instances in one availability zone**
- **Enough instances to survive an AZ loss in count, but not in capacity**
- **No connection pooling**, exhausting the database's connections
- **In-process caches only**, so the hit ratio falls as you scale
- **Local logs**, which disappear with the instance that held the evidence
- **Adding instances** when the database primary is the constraint

---

## 20. Open Source Technologies

- **NGINX**, **HAProxy**, **Envoy**, **Traefik** — load balancing and health checking
- **Kubernetes HPA and KEDA** — scale on custom and queue-based metrics
- **Karpenter**, **cluster-autoscaler** — provision the nodes underneath
- **Redis** — shared sessions and cache
- **PgBouncer** — pool connections so N instances do not exhaust the database
- **Consul**, **etcd** — service discovery and distributed locks for leader election
- **k6**, **Locust** — verify that scaling out actually increases throughput
- **Prometheus**, **Grafana** — per-instance and aggregate metrics

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Kill one instance in staging. If any user notices, you still have state in the application.
- [ ] Check whether sticky sessions are enabled, and why.
- [ ] Find every scheduled job in your application code and confirm only one instance runs it.
- [ ] Calculate instances × pool size and compare with your database's connection limit.
- [ ] Verify your fleet has enough *capacity* — not just instances — to lose an availability zone.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
load balancer → many identical stateless instances across AZs
                        └── shared cache, database, object storage below the line
```

## 2. Request Flow

```text
Input       rising load, and instances that hold nothing unique
    ↓
Processing  a load balancer spreads work; autoscaling adds and removes copies
    ↓
Output      capacity that grows nearly linearly, and survives losing a machine or a zone
```

## 3. Real-World Usage

Horizontal scaling is the default shape of every modern web system, and the interesting part is what it forces: statelessness, shared session storage, idempotent handlers and aggregated observability. Teams usually adopt it for capacity and then discover the availability and deployment benefits were worth more.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Adding more identical instances behind a load balancer |
| **Why does it exist?** | Because one machine has a ceiling and is a single point of failure |
| **Where does it belong?** | The stateless tiers — web, API, workers |
| **When should I use it?** | When one machine is not enough, or when losing one must not matter |

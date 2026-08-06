# High Availability

> **In one line —** designing so that no single failure takes the service down; which means finding every component that exists only once, because those are the only ones that matter.

| | |
|---|---|
| **Category** | Concept / Practice |
| **Architectural Layer** | Cross-cutting |
| **Related notes** | [Disaster Recovery](Disaster%20Recovery.md) · [Load Balancing](Load%20Balancing.md) · [Horizontal Scaling](Horizontal%20Scaling.md) · [Monitoring and Alerting](Monitoring%20and%20Alerting.md) · [Cloud Fundamentals](../12%20-%20Cloud%20Architecture/Cloud%20Fundamentals.md) · [Distributed Systems Fundamentals](../10%20-%20Distributed%20Systems/Distributed%20Systems%20Fundamentals.md) |

---

## 1. Short Definition

*What is it?*

High availability is the property that the service **keeps serving through the failure of any single component**. It is achieved by removing single points of failure: redundancy, automatic detection, and automatic failover.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Everything fails, eventually
    a disk, a process, a machine, a rack, a data centre
    a certificate expires, a deployment is bad, a dependency times out
    ↓
If any one of those stops your service, you do not have a service —
you have a demonstration that happens to be running
```

> [!IMPORTANT]
> **High availability is not achieved by making components reliable; it is achieved by assuming they are not.** The productive question is never "how do we stop this failing?" but "what happens when it does, and how fast do we recover without a human?" Every technique below follows from that reframing.

---

## 3. Architecture Position

```text
   DNS (health-check based failover)          ← is this redundant?
        │
   CDN / edge (many locations)                ← usually yes, by design
        │
   LOAD BALANCER (multi-AZ by default)        ← check this
        │
   ┌────┴─────┬──────────┐
   app        app        app                   ← easy: stateless, N copies
   └────┬─────┴──────────┘
        │
   CACHE (replicated? or a single node?)      ← often a hidden SPOF
        │
   DATABASE primary ──sync──► standby         ← THE hard part
        │
   OBJECT STORAGE (already redundant)
```

> [!TIP]
> **Walk this diagram for your own system and mark everything that exists exactly once.** That list is your availability work, in priority order. It is usually shorter and more mundane than expected — a single cache node, one NAT gateway, one message broker, one certificate with a manual renewal.

---

## 4. What the nines mean

```text
99%       "two nines"    →  3.65 days of downtime per year
99.9%     "three nines"  →  8.8 hours
99.95%                   →  4.4 hours
99.99%    "four nines"   →  53 minutes
99.999%   "five nines"   →  5 minutes
```

```text
Each nine costs roughly an ORDER OF MAGNITUDE more than the last
    99.9%   → multi-AZ, managed services, automated failover
    99.99%  → multi-region, no manual steps anywhere, extensive testing
    99.999% → a large dedicated team, and very few businesses need it
```

> [!CAUTION]
> **Your availability is the product of your dependencies', not the best of them.** Four components each at 99.9% in series give roughly 99.6% — worse than any single one. This is why adding services quietly lowers availability, and why the interesting engineering is in making dependencies *non-fatal* (timeouts, fallbacks, caches, degraded modes) rather than in making each one perfect.

---

## 5. Redundancy patterns

```text
ACTIVE-ACTIVE           all replicas serve traffic
    ✓ no failover delay, capacity fully used
    ✗ needs statelessness or conflict resolution
    → the app tier; ideally the database read path

ACTIVE-PASSIVE           one serves, one waits
    ✓ simple, no conflicts
    ✗ you pay for idle capacity; failover takes time
    → database primaries, single-writer systems

N+1 / N+2                enough spare capacity to lose one or two units
    → the question is CAPACITY, not instance count
```

> [!IMPORTANT]
> **Redundancy in count is not redundancy in capacity.** Three instances each at 70% utilisation cannot absorb the loss of one — the survivors would need 105%. If you run across two availability zones, each zone must be able to serve peak load alone, which means running each at under 50%. Teams discover this during the first real zone failure.

---

## 6. Failure domains

```text
PROCESS      → run more than one process
MACHINE      → more than one machine
RACK         → the provider handles this within an AZ
AVAILABILITY ZONE → separate power, cooling, network
                  → SPAN TWO OR THREE. This is the high-value step.
REGION       → survives a regional event
             → expensive, and forces asynchronous design
PROVIDER     → rarely justified
HUMAN        → the most common cause: review, staging, canaries, rollback
```

> [!TIP]
> **Multi-AZ is where nearly all the availability value is, and it is close to free.** Zones are a few milliseconds apart, so one synchronous system can span them, and managed services do the failover for you. Multi-region is a different discipline entirely — tens or hundreds of milliseconds forces asynchronous replication, conflict resolution and split-brain handling into your application. Do not treat the second as an incremental step from the first.

---

## 7. Detection and failover

```text
DETECTION           health checks, and monitoring that catches "sick but alive"
    ↓
DECISION            automatic, ideally; humans are slow and asleep
    ↓
ACTION              remove from rotation, promote a standby, shift traffic
    ↓
VERIFICATION        did it work? did it fail over cleanly?
```

```text
THE FAILURE MODES OF FAILOVER ITSELF
    SPLIT BRAIN      both nodes think they are primary → data divergence
    FLAPPING         failing back and forth
    UNTESTED PATH    the standby has never served traffic and does not work
    CASCADE          failover shifts load onto something that then also fails
```

> [!CAUTION]
> **Automatic failover that has never been tested is a liability, not a safeguard.** A standby that has never accepted a connection, a promotion script nobody has run, a replica with a subtly different configuration — all of these look like redundancy on a diagram and fail on the day. Test failover deliberately, on a schedule, in production if you can bear it. The teams who do this are the ones whose incidents are brief.

---

## 8. Graceful degradation

```text
A dependency fails. What SHOULD happen?
    ↓
BAD      the whole page returns 500
GOOD     the recommendation panel is hidden; the rest of the page works
```

```text
THE TECHNIQUES
    TIMEOUTS         short and explicit — a hung dependency must not hang you
    CIRCUIT BREAKER  stop calling a failing dependency; fail fast
    FALLBACK         stale cache, default value, or feature hidden
    BULKHEAD         separate connection pools, so one slow dependency
                     cannot consume every thread
    LOAD SHEDDING    reject some requests to keep the rest healthy
    RETRY + BACKOFF  with jitter, and a budget
```

> [!IMPORTANT]
> **The single most common cause of a total outage from a partial failure is a missing timeout.** A dependency that stops answering — rather than refusing — holds your threads or connections until every worker is blocked, and then the whole service is down because one non-essential feature was slow. Every network call needs an explicit timeout shorter than your own, and a decision about what happens when it fires.

---

## 9. Real World Example

- **Multi-AZ RDS** — a synchronous standby with automatic promotion; the highest-value single configuration change available; see [RDS](../12%20-%20Cloud%20Architecture/RDS.md).
- **Auto Scaling Groups replacing failed instances** — self-healing without a human; see [EC2](../12%20-%20Cloud%20Architecture/EC2.md).
- **Kubernetes rescheduling pods** off a lost node; see [Kubernetes](../13%20-%20DevOps%20and%20Delivery/Kubernetes.md).
- **Netflix's chaos engineering** — deliberately terminating instances in production to keep failover paths honest; see [Netflix Architecture](../15%20-%20Real%20World%20System%20Design/Netflix%20Architecture.md).
- **Route 53 health-check failover** between regions, for the rare cases that justify it.
- **A payment provider outage** handled by queueing transactions rather than rejecting them.

---

## 10. Communication and Dependencies

- **[Load Balancing](Load%20Balancing.md)** with health checks — the detection mechanism for the app tier
- **A replicated data store** with automatic promotion
- **At least two availability zones**, with capacity in each
- **[Monitoring and Alerting](Monitoring%20and%20Alerting.md)** — including synthetic checks from outside
- **Automated certificate renewal** — an expired certificate is a common self-inflicted outage
- **Rollback**, since bad deployments cause more downtime than hardware does
- **A rehearsed runbook** for the failures automation does not cover

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Multi-AZ, stateless application tiers, a managed database with automatic failover, and automated instance replacement are worth doing for essentially every production system. They are cheap, well-supported, and cover the failures that actually happen.

> [!CAUTION]
> - **Not multi-region by default** — the complexity is a step change, and it introduces its own outages
> - **Not without testing failover** — untested redundancy is decoration
> - **Not at the cost of understanding** — a system too complex to reason about fails in ways nobody predicted
> - **Not more nines than the business needs** — each one costs an order of magnitude
> - **Not high availability without a backup** — replication is not a backup; see [Backup Strategy](Backup%20Strategy.md)
> - **Not redundancy in count without capacity** to absorb a loss

---

## 12. Advantages and Disadvantages

**Advantages**
- Single failures stop being incidents
- Deployments and maintenance become possible without downtime
- The same properties enable scaling and rolling releases
- Recovery happens without a human, at 4 a.m.
- Confidence to make changes, which compounds into faster delivery

**Disadvantages**
- **Cost** — idle standby capacity, spare headroom, cross-zone traffic
- Complexity, and every added component is another failure mode
- Failover paths need testing, which is real ongoing work
- Split brain and consistency problems that a single node never had
- Availability spent on redundancy is not spent on features
- Can create false confidence: a diagram full of pairs, none of them tested

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Synchronous replication** | 1–2 ms added per commit across AZs; the price of no data loss |
| **Cross-AZ traffic** | Millisecond latency, and charged per GB both ways |
| **Failover window** | Seconds to a couple of minutes — during which you are degraded or down |
| **Required headroom** | Run at under 50% per zone if either must serve alone |
| **Circuit breakers** | Fail fast instead of slowly — a large p99 improvement during incidents |
| **Timeouts** | Convert an outage into a degradation |

> [!TIP]
> **Availability and latency trade against each other, and the choice should be explicit.** Synchronous replication costs milliseconds per write and guarantees no data loss on failover; asynchronous replication is faster and may lose the last few seconds. Neither is correct in general — but the decision belongs to whoever owns the data, not to whoever configured the database.

---

## 14. Security Considerations

> [!CAUTION]
> **Security incidents are availability incidents, and they are the ones redundancy does not help with.** A ransomware event or a compromised credential propagates to your replicas immediately, because replication faithfully copies destruction. Availability architecture protects against components failing; only immutable, isolated backups protect against something deliberately destroying data.

- **Replication is not a backup** — corruption and deletion replicate perfectly
- **Immutable, versioned backups in a separate account** are the availability control for a compromise
- **Expired certificates** cause a striking number of outages — automate renewal and alert 30 days out
- **DDoS protection at the edge**, since availability is what an attacker attacks
- **Rate limiting and load shedding** to keep a service alive under abuse
- **Failover must not weaken controls** — a standby with looser security is a target
- **Runbooks and credentials must be reachable during an outage** — including if your identity provider or wiki is the thing that is down

---

## 15. Mental Model

> [!NOTE]
> **High availability is an aircraft with more than one engine.**
>
> The point is not that the engines never fail; it is that the aircraft flies on one — and that the pilots have practised it in a simulator, repeatedly, so it is procedure rather than improvisation. Notice what this does not protect against: bad fuel loaded into both tanks. Correlated failures and human error bypass redundancy entirely, which is why review, staging and rollback matter as much as duplicate hardware.

---

## 16. Mini Architecture Diagram

```text
                    Route 53 (health checks)
                            │
                    CDN (many edge locations)
                            │
              ┌─────────────▼─────────────┐
              │  LOAD BALANCER, AZ-a+AZ-b │  ← managed, redundant
              └──────┬──────────────┬─────┘
                     │              │
        ┌────────────▼───┐   ┌──────▼─────────┐
        │  AZ-a          │   │  AZ-b          │
        │  app × 2 @45%  │   │  app × 2 @45%  │  ← EITHER zone can
        │                │   │                │    serve peak alone
        │  cache primary │◄──┤  cache replica │
        │       │        │   │                │
        │  DB PRIMARY    │──►│  DB STANDBY    │  sync, auto-promote
        └────────┬───────┘   └────────────────┘
                 │
        object storage (regionally redundant already)
                 │
        ┌────────▼─────────────────────────────┐
        │ IMMUTABLE BACKUPS, separate account  │ ← the ONE thing
        │ (what redundancy cannot give you)    │   replication cannot do
        └──────────────────────────────────────┘

   EVERY external call: timeout · circuit breaker · fallback
```

---

## 17. Complete Request Flow

```text
Normal operation: traffic spread across both zones, database primary in AZ-a
    ↓
─────────────── an instance dies ───────────────
Health check fails twice → removed from rotation in 10 s
    ↓
Remaining instances absorb the load, because headroom existed
    ↓
The scaling group replaces it; nobody is paged
    ↓
─────────────── the database primary fails ───────────────
The managed service detects it and promotes the AZ-b standby
    ↓
The endpoint's DNS is repointed  (60-120 s)
    ↓
Connection pooling held client connections, shortening what users saw
    ↓
No data lost, because replication was synchronous
    ↓
─────────────── an entire AZ fails ───────────────
Half the application capacity disappears; the load balancer routes to AZ-b
    ↓
AZ-b was running at 45%, so it absorbs the full load
    ↓
Database failover as above; autoscaling adds capacity in the surviving zone
    ↓
Degraded in capacity, available throughout
    ↓
─────────────── a dependency degrades ───────────────
The recommendation service stops responding (does not refuse — HANGS)
    ↓
A 500 ms timeout fires instead of threads blocking indefinitely
    ↓
The circuit breaker opens after repeated failures: calls fail instantly
    ↓
Fallback: the panel is hidden; the rest of the page is unaffected
    ↓
One feature is degraded. The site is up. This is the whole point.
    ↓
─────────────── the failure redundancy does NOT cover ───────────────
A bad migration deletes a table at 14:32
    ↓
Replicated instantly to the standby and every replica
    ↓
Redundancy is useless here
    ↓
Point-in-time restore to 14:31 from immutable backups
    ↓
LESSON: HA protects against components failing. Backups protect against data being destroyed.
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> List everything that exists exactly once and fix those first; span two availability zones with enough capacity in each to serve alone; put an explicit timeout, circuit breaker and fallback on every external call; and test failover before the day it matters.

---

## 19. Common Mistakes

- **Redundancy in count but not in capacity** — surviving instances cannot absorb the load
- **Never testing failover**, so the standby's first attempt is during an incident
- **Assuming replication is a backup** — it copies deletions faithfully
- **No timeouts on external calls**, so a hanging dependency takes the whole service down
- **A hidden single point of failure** — one cache node, one NAT gateway, one broker
- **Manual certificate renewal**, which eventually expires at the worst time
- **A deep health check** that removes every backend when a dependency fails
- **Multi-region attempted** before multi-AZ is solid, adding failure modes
- **Chasing more nines** than the business needs or can afford
- **No rollback path**, when bad deployments cause more downtime than hardware
- **Runbooks stored in a system that is also down**
- **Ignoring correlated failure** — both replicas on the same host, same bad config, same expired certificate

---

## 20. Open Source Technologies

- **Patroni**, **repmgr** — PostgreSQL automatic failover with proper leader election
- **Redis Sentinel**, **Redis Cluster** — replication and failover for caches
- **Keepalived / VRRP** — a floating IP for on-premises redundancy
- **etcd**, **Consul**, **ZooKeeper** — consensus and leader election, so failover avoids split brain
- **HAProxy**, **Envoy** — health checking, outlier detection, circuit breaking
- **resilience4j**, **Polly**, **tenacity** — timeouts, retries and circuit breakers in application code
- **Chaos Mesh**, **LitmusChaos**, **Toxiproxy** — inject failures and verify your assumptions
- **Prometheus** + **Alertmanager**, **Blackbox exporter** — detection, including from outside

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] List every component in your system that exists exactly once. That is your work queue.
- [ ] Check your per-zone utilisation: could one zone serve peak alone?
- [ ] Find one external call with no timeout and add one.
- [ ] Trigger a database failover deliberately, in staging, and time it.
- [ ] Confirm certificate renewal is automated and alerted.
- [ ] Write down what your service should do when each dependency is unavailable. Any answer of "return 500" is a design decision to revisit.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
redundant edge → multi-AZ load balancer → app in both zones (each <50%)
                → replicated cache → DB primary + sync standby → immutable backups
```

## 2. Request Flow

```text
Input       a component failing, at an inconvenient time
    ↓
Processing  detected in seconds, traffic routed away, a standby promoted automatically
    ↓
Output      a service that stays up — degraded at worst — with no human involved
```

## 3. Real-World Usage

The systems that stay up are not the ones with the most redundancy; they are the ones whose redundancy has been exercised. Multi-AZ with a managed database, health-checked load balancing, self-healing instance groups and timeouts on every call cover the overwhelming majority of real failures. Beyond that, the largest remaining cause of downtime is not hardware at all — it is change, which is why rollback belongs in this chapter as much as replication does.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Designing so no single component failure takes the service down |
| **Why does it exist?** | Because everything fails, and users do not accept that as an explanation |
| **Where does it belong?** | Every layer — starting with whatever exists only once |
| **When should I use it?** | Multi-AZ and timeouts for every production system; more nines only when justified |

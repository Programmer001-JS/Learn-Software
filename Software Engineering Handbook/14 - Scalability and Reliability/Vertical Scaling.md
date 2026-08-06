# Vertical Scaling

> **In one line —** buying a bigger machine; unfashionable, has a hard ceiling, and is very often the correct next move because it requires no code changes at all.

| | |
|---|---|
| **Category** | Scaling Strategy |
| **Architectural Layer** | Infrastructure |
| **Related notes** | [Scalability Fundamentals](Scalability%20Fundamentals.md) · [Horizontal Scaling](Horizontal%20Scaling.md) · [EC2](../12%20-%20Cloud%20Architecture/EC2.md) · [RDS](../12%20-%20Cloud%20Architecture/RDS.md) · [Performance Engineering](Performance%20Engineering.md) |

---

## 1. Short Definition

*What is it?*

Vertical scaling — scaling *up* — means giving one machine more CPU, memory, disk throughput or network bandwidth, rather than running more machines.

---

## 2. Problem

*What engineering problem does it solve?*

```text
The database is at 90% CPU
    ↓
Option A: shard it
    → months of work, lose joins and transactions, retrain the team
Option B: change the instance type
    → 10 minutes of downtime, 4× the capacity, zero code changes
    ↓
Option B is not a compromise. It is usually the right answer.
```

> [!IMPORTANT]
> **Vertical scaling is the only scaling strategy that requires no architectural change, and that makes it the cheapest capacity you can buy.** The engineering cost of sharding, of introducing a distributed cache, or of making a stateful service stateless is measured in months of the team's time. The cost of a larger instance is measured in dollars and a maintenance window. Compare those honestly before choosing the interesting option.

---

## 3. Architecture Position

```text
   BEFORE                          AFTER
   ┌──────────────┐               ┌────────────────────────┐
   │  4 vCPU      │               │  16 vCPU               │
   │  16 GB RAM   │      →        │  64 GB RAM             │
   │  gp3 3k IOPS │               │  gp3 12k IOPS          │
   └──────────────┘               └────────────────────────┘
   same code · same architecture · same operational model
   same connection string · same everything
```

Nothing above or below changes. That is the entire appeal.

---

## 4. What modern hardware can actually do

```text
A SINGLE CLOUD INSTANCE, TODAY
    up to ~192 vCPU
    up to ~24 TB of RAM (memory-optimised families)
    hundreds of thousands of IOPS on local NVMe
    100 Gbit/s of network
```

```text
What that means in practice
    a PostgreSQL instance can serve tens of thousands of transactions/second
    a working set of hundreds of GB fits entirely in RAM
    most companies never outgrow one large machine for their primary database
```

> [!TIP]
> **The mental image most engineers carry of "one server" is a decade out of date.** Advice to shard early was written when a big machine had 8 cores and 32 GB. Stack Overflow famously served enormous traffic from a handful of servers. Before accepting that you must distribute, check what a single current-generation instance would do — the answer is frequently "all of it, comfortably".

---

## 5. Where the ceiling actually is

```text
THE CEILING IS NOT ALWAYS THE HARDWARE
    single-threaded code       → more cores change nothing
    one process, one lock      → contention, not capacity
    disk-bound queries         → IOPS and throughput, not CPU
    network-bound work         → bandwidth caps per instance size
    licensing per core         → the cost curve turns vertical
```

```text
And the real limits of the approach
    ONE MACHINE = ONE FAILURE DOMAIN
    resizing usually means a RESTART
    price per unit of compute RISES at the top of the range
```

> [!CAUTION]
> **Scaling up does nothing for availability, and that is the honest argument against it.** A larger machine still fails, still needs patching, and still takes your service with it. This is why the mature position is not "vertical or horizontal" but **vertical for capacity, horizontal for redundancy** — two mid-sized instances usually beat one huge one, and a Multi-AZ managed database gives you a large primary *and* a standby.

---

## 6. Sizing: measure what is actually saturated

```text
CPU high?           → more cores, or a faster generation
MEMORY high?        → more RAM; also check for a leak first
DISK slow?          → provisioned IOPS and throughput, or local NVMe
NETWORK capped?     → a larger instance size lifts the cap
NOTHING high?       → the bottleneck is a lock, a query, or a dependency
```

> [!TIP]
> **Before resizing, verify that the resource you are buying is the one that is saturated.** Doubling CPU on a system waiting for disk changes nothing except the bill. Two specific traps: on AWS, `t` family CPU credits and gp3's default IOPS both cause *throttling that looks like ordinary slowness*, and both are fixed by configuration rather than by a larger instance. Check those before anything else.

---

## 7. Vertical scaling as a first move

```text
THE SENSIBLE ORDER
    1. profile — is it a query, a lock, an N+1, a missing index?
    2. fix the obvious inefficiency (usually free, often 10×)
    3. SCALE UP (an afternoon, buys years)
    4. add read replicas (reads only)
    5. cache aggressively
    6. make the app tier horizontal (for availability and elasticity)
    7. shard, only when a single writer genuinely cannot keep up
```

> [!IMPORTANT]
> **Steps 3 and 7 are commonly attempted in the wrong order, and the cost difference is enormous.** Sharding is a year-long project that removes joins, transactions and easy analytics. Scaling up is a maintenance window. Every month spent on the first that could have been bought with the second is a month not spent on the product — and if you scale up first and never need to shard, you have made the correct decision permanently.

---

## 8. What scales vertically particularly well

```text
EXCELLENT FIT
    relational database primaries      ← the classic and best case
    in-memory caches (Redis)          large memory beats many small nodes
    single-writer systems             brokers, coordinators, schedulers
    analytical queries                one large machine, no shuffle overhead
    build machines and monoliths       simply faster

POOR FIT
    stateless web tiers               scale out instead; you also want redundancy
    embarrassingly parallel batch     many cheap machines are cheaper
    anything with an availability SLA  where one machine is unacceptable
```

> [!TIP]
> **A single-node Redis with 200 GB of RAM is simpler and usually faster than a six-node cluster, and it removes an entire class of resharding and hot-key problems.** The same logic applies to analytical work: one big machine with the data in memory avoids the network shuffle that makes distributed queries slow.

---

## 9. Real World Example

- **Stack Overflow** — the canonical case for scaling up: enormous traffic from very few, very large servers.
- **PostgreSQL primaries** — the overwhelming majority of production databases scale up and are never sharded.
- **Redis** — a single large node with Multi-AZ replication, in preference to cluster mode.
- **Monolithic applications** — a bigger instance and an index instead of a microservice migration.
- **Data warehouses and analytics** — DuckDB or ClickHouse on one large machine, outperforming a small cluster.
- **CI build machines**, where the win is per-build speed rather than parallel throughput.

---

## 10. Communication and Dependencies

- **A maintenance window**, since most resizes require a restart
- **Multi-AZ or a standby**, so the restart and the failure domain are covered; see [RDS](../12%20-%20Cloud%20Architecture/RDS.md)
- **Metrics that identify the saturated resource**; see [Monitoring and Alerting](Monitoring%20and%20Alerting.md)
- **Configuration that reflects the new size** — database memory settings, worker counts, JVM heap
- **Reserved capacity or savings plans**, to control the cost of large steady instances

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Scale up first for anything stateful — databases, caches, brokers, single-writer services. It is fast, reversible, needs no code changes, and buys enough time to solve the real problem properly rather than under pressure.

> [!CAUTION]
> - **Not for availability** — one machine is one failure domain, whatever its size
> - **Not when you are already near the largest instance available** — that is the signal to plan differently
> - **Not when the bottleneck is single-threaded** — extra cores will sit idle
> - **Not when per-core licensing** makes the cost curve punitive
> - **Not for stateless tiers**, where scaling out also gives redundancy and elasticity
> - **Not as a substitute for fixing an obviously bad query** — that is paying rent on a bug

---

## 12. Advantages and Disadvantages

**Advantages**
- **No code or architectural changes**
- Effective immediately, and reversible just as easily
- No distributed systems complexity — no partial failure, no consistency puzzles
- Joins, transactions and constraints all keep working
- Simpler to operate, monitor and reason about
- Often better latency than a distributed equivalent, with no network hops
- A very large machine is far cheaper than a year of engineering

**Disadvantages**
- **A hard ceiling** — there is a largest instance
- **No availability benefit** — still one failure domain
- Resizing usually means downtime
- Price per unit of compute rises at the top of the range
- Paying for peak capacity continuously
- Encourages postponing a redesign that may eventually be necessary
- Not every resource scales together in fixed instance families

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **CPU-bound work** | Scales with cores, if the workload is parallel |
| **Single-threaded work** | Only a faster core or newer generation helps |
| **Memory** | The largest wins available — a working set in RAM changes everything |
| **Disk** | Provisioned IOPS or local NVMe; often the real constraint |
| **Network** | Bandwidth caps rise with instance size |
| **NUMA effects** | On very large machines, memory locality starts to matter |
| **Newer generation** | Frequently cheaper *and* faster than more of an older one |

> [!TIP]
> **Try a newer instance generation before a larger size, and try ARM before either.** A current-generation instance is often 15–30% faster per core at a lower price than the previous one, and Graviton-class ARM instances typically offer 20–40% better price-performance for interpreted and JVM workloads. Both are configuration changes with no architectural risk — the cheapest performance work available.

---

## 14. Security Considerations

> [!CAUTION]
> **A larger machine concentrates risk: more data in memory, more tenants on one host, and a single compromise reaching everything.** It also concentrates *availability* risk, which is why vertical scaling should always be paired with a standby or replica — not because the standby serves traffic, but because one machine will eventually fail at an inconvenient moment.

- **Multi-AZ or a replica**, always, for anything scaled up and important
- **Encryption at rest and in transit** — unchanged by size, and still not optional
- **Patching still matters** — a big machine is not fewer vulnerabilities, just fewer hosts
- **Resource limits** so one tenant or query cannot consume the whole machine
- **Backups sized for the machine** — a 4 TB database takes real time to restore; know the number; see [Backup Strategy](Backup%20Strategy.md)
- **Noisy neighbours and side-channel concerns** diminish on the largest sizes, which are often single-tenant hosts

---

## 15. Mental Model

> [!NOTE]
> **Vertical scaling is hiring one excellent specialist; horizontal scaling is hiring a team.**
>
> The specialist needs no coordination, no handover process and no shared documentation — they simply work faster, and everything you already do keeps working. The limit is that there is a best available specialist, and if they are ill nothing happens at all. A team has no ceiling and survives absence, but now you have meetings, coordination and the question of who holds which knowledge. Most organisations should hire the specialist first and build the team when the work genuinely exceeds one person.

---

## 16. Mini Architecture Diagram

```text
   THE PRAGMATIC COMBINATION
   ─────────────────────────────────────────────────────────

              load balancer
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   ┌────────┐  ┌────────┐  ┌────────┐    APP TIER:
   │ app    │  │ app    │  │ app    │    scale OUT
   │ small  │  │ small  │  │ small  │    (redundancy + elasticity)
   └───┬────┘  └───┬────┘  └───┬────┘
       └──────┬────┴───────┬────┘
              ▼            ▼
    ┌───────────────────────────────────┐
    │  DATABASE PRIMARY                  │  DATA TIER:
    │  32 vCPU · 256 GB RAM · 20k IOPS   │  scale UP
    │  working set entirely in memory    │
    └──────────────┬────────────────────┘
                   │ synchronous
    ┌──────────────▼────────────────────┐
    │  STANDBY (same size, other AZ)     │  ← the availability answer
    └───────────────────────────────────┘   to "one machine"
                   │ async
    ┌──────────────▼────────────────────┐
    │  READ REPLICA                      │  ← reads, if needed
    └───────────────────────────────────┘
```

---

## 17. Complete Request Flow

```text
The database is at 90% CPU and p99 has doubled
    ↓
STEP 1 — profile before spending
    pg_stat_statements shows one query is 60% of total time
    A missing index removes it entirely → CPU drops to 45%
    ↓
Six months later, growth returns it to 85%
    ↓
STEP 2 — verify what is saturated
    CPU high, memory fine, disk queue shallow → genuinely CPU-bound
    ↓
STEP 3 — newer generation first
    Move from the previous generation to the current one, same size
    → 25% more throughput, slightly lower price, no downtime beyond a failover
    ↓
A year later, growth returns
    ↓
STEP 4 — scale up
    Double the instance size during a maintenance window
    Multi-AZ makes it a failover of ~60 seconds, not an outage
    Adjust memory settings for the new size
    → 4× headroom for one afternoon of work
    ↓
STEP 5 — offload what does not need the primary
    Reporting moved to a read replica
    Hot queries cached
    ↓
─────────────── three years in ───────────────
Approaching the largest available instance; writes are the constraint
    ↓
NOW sharding is the right project — with three years of revenue,
a bigger team, real usage data to choose a shard key,
and time to do it properly
    ↓
─────────────── the counterfactual ───────────────
Sharding at step 1 would have cost a year,
removed joins and transactions,
and the actual problem was a missing index.
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Scale up first for stateful systems — it needs no code changes and modern machines are far bigger than most people assume — but always pair it with a standby, because a larger machine is still one failure domain.

---

## 19. Common Mistakes

- **Dismissing vertical scaling as unfashionable** and going straight to sharding
- **Resizing before profiling**, and buying more of a resource that was not the constraint
- **Ignoring `t` family CPU credits and gp3 IOPS defaults**, mistaking throttling for load
- **Adding cores to single-threaded work**
- **Scaling up without a standby**, so the whole system is one machine
- **Not adjusting configuration** after resizing — a database with the old memory settings gains little
- **Trying a larger size before a newer generation or ARM**, which are often cheaper and faster
- **Paying on-demand rates** for a large, steady instance instead of reserving it
- **Scaling up the stateless tier**, where scaling out also buys redundancy
- **Assuming there is no ceiling**, and discovering the largest instance during an incident
- **Never measuring restore time** for a very large database

---

## 20. Open Source Technologies

- **PostgreSQL** — scales up remarkably well; tune `shared_buffers` and `work_mem` after resizing
- **Redis** — one large node beats a small cluster for most workloads
- **DuckDB**, **ClickHouse** — analytics on a single large machine, often beating a cluster
- **perf**, **flamegraph**, **py-spy**, **async-profiler** — find out whether you are actually CPU-bound
- **pg_stat_statements**, **pgbadger** — the queries to fix before buying hardware
- **fio** — measure real disk throughput and IOPS before blaming the database
- **htop**, **iostat**, **sar** — the basics that answer "which resource is saturated"
- **hyperfine**, **pgbench**, **sysbench** — benchmark before and after a resize

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Look up the largest instance type available for your database. Compare it with what you run now.
- [ ] Identify which resource is actually saturated on your busiest machine.
- [ ] Check whether a newer generation or ARM equivalent would be faster and cheaper.
- [ ] Confirm anything you have scaled up has a standby in another zone.
- [ ] Time a restore of your largest database. Write the number down.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
one larger machine (data tier, scale UP) + a standby in another AZ
                 beneath a horizontally scaled stateless app tier
```

## 2. Request Flow

```text
Input       a saturated resource on one machine
    ↓
Processing  profile, fix the obvious inefficiency, then buy more of the right resource
    ↓
Output      multiples more capacity with no architectural change — and the same failure domain
```

## 3. Real-World Usage

The quiet reality of production systems is that most databases are never sharded: they are indexed, cached, given a read replica, and moved onto a larger instance every couple of years. Modern hardware absorbed a decade of growth that older advice assumed would force distribution — and the companies that scaled up first generally spent their engineering time on their product instead.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Giving one machine more CPU, memory, disk or network |
| **Why does it exist?** | Because capacity you can buy is cheaper than capacity you must design |
| **Where does it belong?** | The stateful tier — databases, caches, brokers, single writers |
| **When should I use it?** | First, after fixing obvious inefficiency — and always with a standby |

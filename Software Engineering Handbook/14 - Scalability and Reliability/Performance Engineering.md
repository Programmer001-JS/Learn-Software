# Performance Engineering

> **In one line —** making systems fast by measuring first; the bottleneck is essentially never where you assumed, and the fix is essentially always cheaper than the redesign you were considering.

| | |
|---|---|
| **Category** | Concept / Practice |
| **Architectural Layer** | Cross-cutting |
| **Related notes** | [Load Testing](Load%20Testing.md) · [Scalability Fundamentals](Scalability%20Fundamentals.md) · [Caching](../08%20-%20Databases%20and%20Data/Caching.md) · [Database Indexing](../08%20-%20Databases%20and%20Data/Database%20Indexing.md) · [Monitoring and Alerting](Monitoring%20and%20Alerting.md) · [Vertical Scaling](Vertical%20Scaling.md) |

---

## 1. Short Definition

*What is it?*

Performance engineering is the disciplined loop of **measure → find the dominant cost → fix it → measure again**. It is distinguished from "optimisation" by the measuring: without it, you are guessing expensively.

---

## 2. Problem

*What engineering problem does it solve?*

```text
"The application is slow"
    ↓
The team spends two weeks optimising the JSON serialiser
    ↓
No improvement
    ↓
The actual cause: one endpoint issuing 400 database queries in a loop
    ↓
Fixed in an hour, once someone looked
```

> [!IMPORTANT]
> **Performance work without measurement is not engineering, it is superstition — and expensive superstition.** Human intuition about where time is spent is reliably wrong, because the costly parts are the ones you are not thinking about: an N+1 query, a missing index, a synchronous call to a third party, an unbounded result set. A profiler answers in ten minutes what a team can argue about for a fortnight.

---

## 3. Architecture Position

```text
   WHERE TIME ACTUALLY GOES, in a typical web request
   ─────────────────────────────────────────────────────
   network + TLS         5-50 ms    (fix with a CDN, keep-alive, HTTP/2)
   load balancer         1-5 ms
   application CPU       1-20 ms    ← where people look first
   DATABASE QUERIES      10-500 ms  ← where the time usually IS
   external API calls    50-2000 ms ← and here
   serialisation         1-10 ms
   ─────────────────────────────────────────────────────
   FRONTEND rendering    often MORE than all of the above
```

> [!TIP]
> **Start at the layer with the largest numbers, and be honest that it is usually not your code.** Two places dominate real systems: the database and calls to other services. A third, frequently ignored by backend engineers, is the browser — where a 200 ms API response sits inside a 4-second page load caused by unoptimised images and blocking JavaScript.

---

## 4. Latency numbers worth memorising

```text
L1 cache reference               ~1 ns
main memory reference           ~100 ns
SSD random read                 ~100 µs      (100,000 ns)
same-datacentre round trip      ~500 µs
disk seek (spinning)            ~10 ms
network, same region            ~1 ms
network, cross-continent        ~100 ms
```

```text
The consequence
    memory is ~100× faster than SSD
    SSD is ~100× faster than a cross-region network call
    → ONE avoided network call beats a thousand micro-optimisations
```

> [!IMPORTANT]
> **These orders of magnitude are the whole basis of performance intuition.** They explain why caching works, why an N+1 query is catastrophic while a slightly inefficient loop is not, and why moving a call from cross-region to in-region can outperform any amount of code tuning. Internalise the ratios and most performance decisions become obvious.

---

## 5. Amdahl's law and the profiler's verdict

```text
If a function is 5% of total time, making it INFINITELY fast
saves 5%.
```

```text
THE ONLY QUESTION WORTH ASKING
    what is the largest single contributor to the time?
    ↓
Fix that. Then ask again — the answer will have changed.
```

> [!CAUTION]
> **Optimising something that is not the dominant cost is worse than doing nothing, because it also adds complexity.** A clever cache, a hand-rolled algorithm or an unsafe concurrency trick that saves 2% is a permanent maintenance cost for a rounding error. If a profile does not show a component as significant, leave it alone — no matter how obviously improvable it looks.

---

## 6. The database, where the time actually is

```text
THE USUAL SUSPECTS, in order of frequency
    N+1 QUERIES        one query, then one per row  → 1 + 400 round trips
    MISSING INDEX      a sequential scan of millions of rows
    SELECT *           fetching columns nobody reads, including large text
    NO PAGINATION      an unbounded result set that grows with your success
    CHATTY ORM         a lazy-loaded association inside a loop
    LOCK CONTENTION    a long transaction blocking everything behind it
    NO CONNECTION POOL a fresh connection per request
```

```text
THE TOOLS THAT FIND THEM
    EXPLAIN ANALYZE           what the planner actually did
    pg_stat_statements        which query consumes the most total time
    slow query log            everything above a threshold
```

> [!TIP]
> **`pg_stat_statements` ordered by total time is the single highest-value diagnostic in most systems.** It surfaces not the slowest query but the one consuming the most database time overall — often a 12 ms query executed 40,000 times a minute, which no slow-query log would ever show. Fix in that order and the improvements are dramatic.

---

## 7. Percentiles, not averages

```text
1,000 requests: 999 at 10 ms, one at 10 seconds
    average          20 ms      "excellent!"
    p50              10 ms
    p99             ~10 s       ← one user in a hundred had an awful time
```

```text
WHY THE TAIL MATTERS MORE THAN IT LOOKS
    a page making 20 API calls hits the p99 of at least one, usually
    → the p99 of a component becomes the p50 of a page
```

> [!IMPORTANT]
> **Averages are actively misleading and should be removed from your dashboards.** A system's felt quality is its tail: p99 is what generates complaints, abandoned carts and support tickets. And in composed systems the tail compounds — a request touching ten services will frequently encounter one service's slow path, which is why tail latency is a system property rather than a per-service nicety.

---

## 8. The fixes, in order of value per unit of effort

```text
1. ADD AN INDEX                  minutes, often 100×
2. FIX THE N+1                   an hour, often 10-50×
3. CACHE THE HOT PATH            hours, 10× on reads
4. PAGINATE                      converts unbounded into bounded
5. MOVE WORK OFF THE REQUEST     queue it; latency becomes someone else's
6. BATCH                         one round trip instead of 100
7. CDN THE STATIC ASSETS         removes most requests entirely
8. SCALE UP                      an afternoon; see Vertical Scaling
9. OPTIMISE THE CODE             days, usually a few percent
10. REWRITE IN A FASTER LANGUAGE months, and it was the database anyway
```

> [!TIP]
> **This list is roughly ordered by return on effort, and most teams start near the bottom.** The rewrite is attractive because it is interesting; the index is boring and does more. Work down the list, measuring after each step, and stop when the system is fast enough — "fast enough" being defined by an SLO, not by aesthetics.

---

## 9. Real World Example

- **A missing index** on a foreign key turning a 4-second page into 40 ms — the most common single fix in the industry.
- **An N+1 in an ORM-heavy admin page**, resolved by eager loading.
- **A synchronous third-party call** in the checkout path, moved to a queue with a fallback.
- **Frontend performance** — image sizing, code splitting and lazy loading routinely outweighing all backend work; see [Web Performance](../05%20-%20Frontend%20Architecture/Web%20Performance.md).
- **A `SELECT *` returning a large text column** on a list endpoint, transferring megabytes nobody read.
- **Connection pool exhaustion** presenting as "the database is slow" when the database was idle.

---

## 10. Communication and Dependencies

- **Profilers** for your language, usable in production
- **Distributed tracing** — otherwise you cannot tell which service consumed the time
- **Database statistics** — `pg_stat_statements`, slow query logs, `EXPLAIN`
- **Percentile metrics**, not averages; see [Monitoring and Alerting](Monitoring%20and%20Alerting.md)
- **A load testing setup** to reproduce the condition; see [Load Testing](Load%20Testing.md)
- **A realistic dataset** — performance problems only appear at production scale
- **A performance budget**, so "fast enough" has a definition

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Do performance work when a measurement shows a user-visible problem, or when a load test reveals a bottleneck you will hit soon. Establish percentile metrics early, so you can tell when something regressed and by how much.

> [!CAUTION]
> - **Not without measuring first** — you will optimise the wrong thing
> - **Not in a development environment** with 100 rows; the plan changes with data volume
> - **Not by micro-optimising** what the profiler shows as insignificant
> - **Not at the cost of clarity** for a few percent
> - **Not by rewriting in a faster language** before checking whether the database is the constraint
> - **Not endlessly** — hit the budget and move on
> - **Not by adding a cache** to hide a query that should simply be indexed

---

## 12. Advantages and Disadvantages

**Advantages**
- Directly improves what users experience, and often conversion
- Reduces infrastructure cost as a side effect
- Postpones or removes the need for scaling work entirely
- Profiling reveals genuine bugs, not only slowness
- Fast systems are more debuggable and more predictable under load
- The early fixes are cheap and enormous

**Disadvantages**
- **Optimised code is frequently less readable**
- Caches introduce staleness and invalidation bugs
- Denormalisation trades write complexity and consistency for read speed
- Measurement infrastructure has its own cost
- Easy to spend weeks for a few percent, once the cheap wins are gone
- Production-only problems are hard to reproduce safely

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Index added** | 10–1000× on the affected query |
| **N+1 removed** | 10–50× on that endpoint |
| **Cache hit** | Microseconds versus milliseconds — the biggest single lever |
| **Connection pooling** | Removes handshake cost and prevents exhaustion |
| **Batching** | One round trip instead of N — the ratio *is* the improvement |
| **HTTP/2 + keep-alive** | Removes per-request handshakes |
| **Compression** | Less bandwidth, slightly more CPU; almost always worth it |
| **Utilisation above ~80%** | Latency rises non-linearly; headroom is a performance feature |

> [!CAUTION]
> **A system at 85% utilisation is not efficient, it is on a cliff edge.** Queueing theory says latency climbs steeply as utilisation approaches saturation, which is why systems appear healthy and then collapse suddenly. If your p99 is bad while averages look fine, suspect queueing before suspecting code.

---

## 14. Security Considerations

> [!CAUTION]
> **Performance work has two specific security traps: caches that ignore authorisation, and timing that leaks information.** A cache keyed only on a URL will serve one user's data to another, which is a data breach caused by an optimisation. And comparing secrets with a normal string comparison leaks their contents through timing — use constant-time comparison for tokens and signatures.

- **Cache keys must include the authorisation context** — user, tenant, role
- **Constant-time comparison** for secrets, tokens and signatures
- **Bound every result set** — unbounded pagination is a denial-of-service vector
- **Rate limit expensive endpoints**; an attacker will find your costliest query
- **Query timeouts**, so one pathological request cannot hold resources indefinitely
- **Profilers and debug endpoints must not be exposed** in production
- **Detailed timing in error messages** can reveal internal structure

---

## 15. Mental Model

> [!NOTE]
> **Performance work is finding the traffic jam, not buying a faster car.**
>
> A journey taking two hours is not improved by a car that goes 20% faster if ninety minutes were spent stationary at one junction. Measure where the time went, fix the junction, then measure again — because the new bottleneck will be somewhere else entirely. And there is a point where the road is clear and the remaining gains require a helicopter: that is where you stop, because "fast enough" is a real destination.

---

## 16. Mini Architecture Diagram

```text
   1. MEASURE — where does the time go?
   ┌────────────────────────────────────────────────────────┐
   │  distributed trace of ONE slow request                  │
   │  ┌──────────────────────────────────────────────────┐  │
   │  │ LB 2 ms │ app 8 ms │ DB 780 ms │ ext API 120 ms  │  │
   │  └──────────────────────────────────────────────────┘  │
   │                       ▲                                 │
   │                   90% here                              │
   └────────────────────────┬───────────────────────────────┘
                            ▼
   2. DRILL IN — pg_stat_statements ordered by TOTAL time
        query A: 12 ms × 40,000/min  ← the real cost
        query B: 900 ms × 3/min      ← looks worse, matters less
                            ▼
   3. DIAGNOSE — EXPLAIN ANALYZE
        Seq Scan on orders (rows=4,000,000)  ← missing index
                            ▼
   4. FIX — CREATE INDEX ... (minutes)
                            ▼
   5. MEASURE AGAIN — p99 from 1.4 s to 90 ms
                            ▼
   6. ASK AGAIN — the bottleneck is now the external API
        → move it off the request path, into a queue
                            ▼
   7. STOP when the performance budget is met
```

---

## 17. Complete Request Flow

```text
Report: "the orders page is slow"
    ↓
STEP 1 — quantify. p50 = 400 ms, p99 = 6 s. Not "slow" — a TAIL problem.
    ↓
STEP 2 — trace one slow request
    app CPU 12 ms · database 5,900 ms · everything else negligible
    ↓
STEP 3 — pg_stat_statements by total time
    One query, 14 ms each, executed 380 times PER REQUEST
    → a textbook N+1 from a lazy-loaded association in a loop
    ↓
STEP 4 — fix with eager loading: 380 queries become 2
    p99 drops from 6 s to 700 ms   (8×, one hour of work)
    ↓
STEP 5 — measure again. The remaining 700 ms is one query.
    EXPLAIN ANALYZE: sequential scan over 4 million rows
    ↓
STEP 6 — add an index on the filtered column
    p99 drops to 120 ms   (another 6×, ten minutes of work)
    ↓
STEP 7 — measure again. Now dominated by a third-party enrichment call.
    Moved to a background job; the page renders without it
    p99 = 45 ms
    ↓
TOTAL: 6 s → 45 ms in one day, with no rewrite and no new infrastructure
    ↓
─────────────── the counterfactual ───────────────
The team's original plan was to rewrite the service in a faster language
    ↓
Three months. The database work would have been identical afterwards.
    ↓
─────────────── the frontend postscript ───────────────
The API now responds in 45 ms; the page still takes 4.2 s to be usable
    ↓
Unsized hero images and a blocking analytics script
    ↓
Fixed in an afternoon: 4.2 s → 1.1 s
    ↓
LESSON: measure what the USER experiences, not what your service reports
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Measure before changing anything, fix the single largest contributor, then measure again — and expect the answer to be a missing index, an N+1, or a synchronous external call, not your code.

---

## 19. Common Mistakes

- **Optimising without profiling**, and improving something irrelevant
- **Testing with 100 rows**, where the query plan differs from production
- **Watching averages** while the p99 is unacceptable
- **Rewriting in a faster language** when the database was the constraint
- **Caching to hide a missing index**, adding staleness for no reason
- **A cache key without the user or tenant**, leaking data between users
- **`SELECT *`** dragging large columns across the network
- **Unbounded queries** that work until the table grows
- **Micro-optimising** what the profiler shows as 2%
- **No connection pooling**, then blaming the database
- **Running at 85% utilisation** and treating the resulting latency as a code problem
- **Ignoring the frontend**, where the user's time is often mostly spent
- **No performance budget**, so the work never ends
- **Debug and profiling endpoints exposed** in production

---

## 20. Open Source Technologies

- **perf**, **flamegraph** — system-wide profiling and the clearest visualisation of where time goes
- **py-spy**, **async-profiler**, **pprof**, **clinic.js** — language-specific production profilers
- **pg_stat_statements**, **pgbadger**, **EXPLAIN ANALYZE** — start here for most systems
- **OpenTelemetry**, **Jaeger**, **Tempo** — find the slow hop in a distributed request
- **k6**, **Locust**, **wrk**, **vegeta** — reproduce load; see [Load Testing](Load%20Testing.md)
- **Lighthouse**, **WebPageTest** — the frontend half nobody profiles
- **hyperfine**, **pgbench**, **sysbench** — before-and-after benchmarks
- **fio**, **iostat**, **htop** — confirm which resource is actually saturated
- **Redis**, **Varnish** — the caches you add once you know what to cache

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Open `pg_stat_statements` ordered by total time. Explain the top entry.
- [ ] Run `EXPLAIN ANALYZE` on your slowest endpoint's main query. Look for a sequential scan.
- [ ] Count the queries your busiest endpoint issues. If it scales with row count, you have an N+1.
- [ ] Compare your p50 and p99. A large gap means queueing, not slow code.
- [ ] Run Lighthouse against your own product and see how the backend time compares with the total.
- [ ] Write down a performance budget for one critical journey.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
trace the request → find the dominant layer → drill into it → fix → measure again
                          (usually: database, then external calls, then frontend)
```

## 2. Request Flow

```text
Input       a slow user-visible operation, quantified in percentiles
    ↓
Processing  profile, identify the largest single cost, fix it, re-measure
    ↓
Output      a system fast enough against a defined budget, with less complexity than a rewrite
```

## 3. Real-World Usage

The distribution of real performance fixes is remarkably consistent across the industry: indexes, N+1 queries, missing pagination, synchronous external calls, and frontend asset weight. The exotic work — algorithmic rewrites, custom serialisers, alternative runtimes — is real but rare, and almost always comes after the list above has been exhausted.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The measure-fix-measure loop for making systems fast |
| **Why does it exist?** | Because intuition about where time goes is reliably wrong |
| **Where does it belong?** | Wherever the profile points — usually the data layer and the network |
| **When should I use it?** | When a measurement shows user-visible slowness; and stop at the budget |

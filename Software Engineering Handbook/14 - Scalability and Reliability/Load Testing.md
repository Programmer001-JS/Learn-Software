# Load Testing

> **In one line —** finding out where your system breaks before your users do; and the result is worthless unless the test resembles real traffic against real data volumes.

| | |
|---|---|
| **Category** | Testing / Practice |
| **Architectural Layer** | Delivery |
| **Related notes** | [Performance Engineering](Performance%20Engineering.md) · [Scalability Fundamentals](Scalability%20Fundamentals.md) · [Monitoring and Alerting](Monitoring%20and%20Alerting.md) · [Horizontal Scaling](Horizontal%20Scaling.md) · [CI CD](../13%20-%20DevOps%20and%20Delivery/CI%20CD.md) |

---

## 1. Short Definition

*What is it?*

Load testing generates **synthetic traffic against a system to observe its behaviour under load**: where latency degrades, what saturates first, and at what point it stops working. The output is a number and a bottleneck, not a pass or a fail.

---

## 2. Problem

*What engineering problem does it solve?*

```text
The system works fine with 50 users
    ↓
A marketing campaign brings 5,000
    ↓
Nobody knows what happens at 5,000 — so you find out in production
    ↓
And the failure is never graceful:
    connection pool exhausted → every request fails
    memory grows → OOM → restart → cold cache → worse
    a queue backs up → work never catches up
```

> [!IMPORTANT]
> **Load testing exists to answer three specific questions, and vague performance curiosity is not one of them.** *What is our maximum throughput? What breaks first? How does it behave beyond that point?* The third is the most valuable and the most neglected: systems rarely slow down gracefully, they fall over, and knowing the shape of that failure is what lets you add the timeout, limit or shed-load rule that turns a collapse into a degradation.

---

## 3. Architecture Position

```text
   load generator (not on the same machine!)
        │  realistic scenarios, realistic think time
        ▼
   ┌─── ENVIRONMENT UNDER TEST ────────────────┐
   │  production-LIKE size and data volume      │
   │  load balancer → app tier → DB + cache     │
   └────────────────┬──────────────────────────┘
                    │
   OBSERVE EVERYTHING WHILE IT RUNS
     client side: throughput, p50/p95/p99, error rate
     server side: CPU, memory, connections, queue depth,
                  DB query times, GC pauses, saturation
```

> [!TIP]
> **A load test without server-side observability tells you *that* it broke, not *why*.** The client's numbers are the symptom; the value is in the correlation with connection counts, query times and GC pauses. If you can only do one thing to improve your load testing, instrument the target rather than refining the script.

---

## 4. The types, and what each is for

```text
LOAD TEST        expected peak traffic
                 → "does it meet the SLO at the load we expect?"

STRESS TEST      increase until it breaks
                 → "where is the ceiling, and what fails first?"

SPIKE TEST       0 → peak instantly
                 → "does autoscaling arrive in time? do caches survive?"

SOAK TEST        moderate load for HOURS
                 → memory leaks, connection leaks, disk filling, log growth

BREAKPOINT       ramp steadily to failure
                 → the curve, not just the number
```

> [!CAUTION]
> **The soak test is the one people skip, and it catches a distinct class of bug nothing else does.** Memory leaks, unclosed connections, growing log files, fragmenting caches and slow queries that degrade as tables grow are invisible in a ten-minute run. An eight-hour soak at moderate load has found more production incidents in advance than any peak-load test.

---

## 5. Realism is the whole game

```text
A USELESS TEST                        A USEFUL TEST
hammers one endpoint                   a realistic MIX of endpoints
1,000 identical requests               varied parameters and users
no think time                          pauses like a human
empty database                         production-SCALE data volume
warm cache, one hot key                realistic cache hit ratio
one user's credentials                 many distinct users and tenants
```

> [!IMPORTANT]
> **Data volume is the single most common reason a load test passes and production fails.** A query that scans a table is instant against 1,000 rows and fatal against 10 million, because the planner chooses a different strategy. If your test database is small, you have tested the wrong system — and the same applies to cache behaviour: a test hitting one hot key reports a 100% hit rate that real traffic will never achieve.

> [!TIP]
> **Derive the scenario mix from your access logs.** Real traffic distributions are surprising — usually a small number of endpoints dominate, and one of them is something nobody would have guessed. Guessing the mix produces a test that exercises the wrong code.

---

## 6. Concurrency, not requests per second

```text
"1,000 requests per second" is ambiguous.
    1,000 rps at 50 ms each   →  50 concurrent requests
    1,000 rps at 2 s each     →  2,000 concurrent requests

CONCURRENCY is what consumes resources:
    threads, connections, memory, database connections
```

```text
LITTLE'S LAW      concurrency = throughput × latency
    → if latency rises under load, concurrency rises too, which raises
      latency further. That feedback loop is how systems collapse.
```

> [!CAUTION]
> **Closed-model load tests (fixed number of virtual users) hide overload; open-model tests (fixed arrival rate) expose it.** With fixed users, slow responses automatically reduce the request rate — the test politely backs off exactly as real traffic would not. Real users keep arriving. Use an arrival-rate model for anything where you care about behaviour beyond capacity.

---

## 7. Reading the results

```text
HEALTHY                          SATURATED
throughput rises with load        throughput FLAT despite more load
latency roughly stable            latency climbing steeply
error rate ~0                     errors appearing
                                  → you have passed the knee of the curve
```

```text
THE KNEE IS YOUR CAPACITY
    → and run production at 50-70% of it, not 90%
```

> [!TIP]
> **The number you want is the throughput at which latency starts climbing, not the throughput at which errors start.** Between those two points the system is technically working and users are unhappy — and that band is where most production incidents live. Report the knee, not the breaking point.

---

## 8. Where to run it

```text
PRODUCTION             the only fully accurate answer, and the riskiest
                       → do it during low traffic, with a kill switch,
                         and with a plan for the data you create

PRODUCTION-LIKE STAGING  the practical default
                       → same instance types, same data VOLUME, same config
                       → a half-sized environment gives half-useful answers

SMALL STAGING          fine for finding N+1s and leaks, useless for capacity

IN CI                  a short smoke-level test to catch REGRESSIONS
                       → not capacity, but "did this release get slower?"
```

> [!CAUTION]
> **Never point a load generator at a third party without permission.** Payment providers, email services and partner APIs will see it as an attack, and rate limits, bans or a bill may follow. Stub external dependencies — but then remember your test no longer measures their latency, which is often a real part of your p99.

---

## 9. Real World Example

- **Before a known event** — Black Friday, a product launch, a marketing campaign with a known audience size.
- **Capacity planning** — establishing how many instances a given traffic level needs.
- **Validating autoscaling** — a spike test proving that new capacity arrives before users notice.
- **Regression detection in CI** — a short test that fails the build if p95 degrades beyond a threshold.
- **Soak testing before a long weekend**, hunting the leak that only appears after six hours.
- **Sizing a new database instance** by replaying real query patterns at multiples of current volume.

---

## 10. Communication and Dependencies

- **A production-like environment**, especially in data volume
- **Server-side observability during the run** — otherwise the results are uninterpretable
- **Realistic scenarios**, derived from access logs
- **Test data generation** at scale, including many distinct users
- **A load generator that is not itself the bottleneck** — and not on the target machine
- **Stubs for third parties**, with their latency simulated rather than removed
- **A cleanup plan** for the data a test creates

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Load test before any event with predictable traffic, when validating autoscaling, when sizing infrastructure, and as a small regression check in CI. Once you know your knee, re-establish it after significant architectural change.

> [!CAUTION]
> - **Not against an empty or small database** — you will test the wrong query plans
> - **Not against third parties** without permission
> - **Not without server-side metrics** — you will learn nothing actionable
> - **Not as a substitute for profiling** — a load test finds the bottleneck's existence, a profiler finds its cause
> - **Not with unrealistic scenarios**, which produce confident wrong answers
> - **Not from one machine** if you need serious throughput; the generator saturates first
> - **Not as a one-off** — the result expires with the next architectural change

---

## 12. Advantages and Disadvantages

**Advantages**
- Turns "will it hold?" into a number
- Reveals the failure mode, which is what you actually need to defend against
- Validates autoscaling and rate limiting before they matter
- Catches leaks and cumulative degradation via soak tests
- Enables capacity planning with evidence instead of guesswork
- In CI, catches performance regressions at review time

**Disadvantages**
- **A realistic environment is expensive** to build and maintain
- Test data generation at scale is real work
- Results are only as good as the scenario's realism
- Third-party dependencies must be stubbed, changing the measurement
- Testing in production carries genuine risk
- Results go stale as the system changes
- Easy to produce impressive numbers that mean nothing

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **The generator** | Needs its own capacity; a saturated generator reports false latency |
| **Network between them** | Same-region for capacity numbers; cross-region adds noise |
| **Coordinated omission** | Many tools under-report tail latency — a well-known measurement bug |
| **Warm-up** | Discard the first minute: JIT, caches and pools need to settle |
| **Test duration** | Minutes for capacity, hours for leaks |
| **Cache state** | Realistic hit ratio, or your results are meaningless |

> [!CAUTION]
> **Coordinated omission is why many load test results understate tail latency badly.** If the tool waits for a slow response before sending the next request, the requests it *did not* send during the stall are never counted as slow — so a two-second stall vanishes from the percentiles. Use tools that correct for it (`wrk2`, `k6` with arrival-rate executors, HdrHistogram-based tooling) if the tail matters, which it does.

---

## 14. Security Considerations

> [!CAUTION]
> **A load test is technically indistinguishable from a denial-of-service attack, so treat it as a coordinated activity rather than something you start quietly.** Notify your provider if required, warn your own on-call so nobody responds to a self-inflicted incident, and never test infrastructure you do not own.

- **Written authorisation** for anything not entirely yours
- **Warn on-call and silence alerts** for the window, then re-enable them
- **Never use production personal data** in test scenarios — generate synthetic users
- **A kill switch**, and someone watching it, for production tests
- **Clean up the data created** — orders, accounts and emails a test generated
- **Ensure the test cannot send real email, SMS or payments**
- **Load testing reveals your rate limits and capacity** — the results are sensitive information

---

## 15. Mental Model

> [!NOTE]
> **A load test is a bridge's load rating, established by driving progressively heavier lorries across it.**
>
> You are not proving it holds one lorry; you are finding the weight at which it starts to flex, and then posting a limit comfortably below that. The test only means something if the lorries resemble real traffic — testing with one lorry driven back and forth tells you nothing about rush hour. And a rating established before the bridge was extended is no longer a rating.

---

## 16. Mini Architecture Diagram

```text
   ┌── LOAD GENERATORS (own capacity, same region) ──┐
   │  scenario mix derived from access logs:          │
   │    60% browse · 25% search · 10% cart · 5% pay   │
   │  arrival-rate model (open), realistic think time │
   │  many distinct users and tenants                  │
   └──────────────────┬──────────────────────────────┘
                      │  ramp: 100 → 200 → 400 → 800 rps
                      ▼
   ┌──── ENVIRONMENT UNDER TEST (production-LIKE) ────────┐
   │  same instance types · SAME DATA VOLUME · same config│
   │                                                       │
   │  LB ──► app tier ──► cache ──► database              │
   │                                                       │
   │  OBSERVED THROUGHOUT:                                 │
   │    CPU · memory · GC pauses                           │
   │    DB connections ← usually the first ceiling         │
   │    query times · cache hit ratio · queue depth        │
   └───────────────────────────────────────────────────────┘
                      │
                      ▼
   RESULT
     throughput plateaus at 620 rps
     latency knee at 480 rps      ← THIS is your capacity
     first thing to saturate: database connections
     → run production at 250-350 rps per unit, and pool properly
```

---

## 17. Complete Request Flow

```text
GOAL: a campaign is expected to bring 5× normal traffic next month
    ↓
STEP 1 — build a realistic scenario from access logs
    the mix is not what anyone guessed: one search endpoint is 25% of traffic
    ↓
STEP 2 — restore a production-sized dataset into staging
    (an earlier test against 10,000 rows had been passing happily)
    ↓
STEP 3 — ramp test, arrival-rate model, server-side metrics recording
    ↓
100 rps → p99 80 ms, all healthy
200 rps → p99 95 ms
400 rps → p99 340 ms   ← the knee; latency is climbing faster than load
500 rps → p99 2.1 s, errors appear
550 rps → throughput STOPS rising; requests fail
    ↓
STEP 4 — what broke? Not CPU (45%), not memory.
    Database connections at the pool limit; requests queueing for one
    ↓
STEP 5 — fix: a connection pooler, and a bounded pool per instance
    Retest: knee moves to 900 rps, ceiling to 1,400
    ↓
STEP 6 — spike test: 0 → 900 rps instantly
    Autoscaling takes 4 minutes; the first 90 seconds return errors
    → pre-warm before the campaign, and raise the minimum instance count
    ↓
STEP 7 — soak test, 6 hours at 300 rps
    Memory grows steadily → an unclosed HTTP client leaking connections
    → would have caused a restart loop on the campaign's second day
    ↓
RESULT: capacity known, two real bugs fixed, autoscaling adjusted
    ↓
─────────────── the campaign ───────────────
Peak traffic 780 rps. Below the 900 knee. p99 stayed at 140 ms.
    ↓
─────────────── the version without testing ───────────────
Connection pool exhaustion at 500 rps, 20 minutes into the campaign
    ↓
Every request failing, cold caches after each restart, and a leak
that would have surfaced the next day anyway
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Test with production-scale data and a realistic scenario mix, use an arrival-rate model so overload is visible, watch server-side metrics to learn *what* saturated, report the latency knee rather than the breaking point, and run a soak test for the bugs nothing else finds.

---

## 19. Common Mistakes

- **A small test database**, so query plans differ from production
- **Hammering one endpoint** instead of a realistic mix
- **No think time**, producing unrealistic concurrency
- **One hot cache key**, reporting a hit ratio real traffic never reaches
- **No server-side metrics**, so you know it broke but not why
- **A closed model** (fixed virtual users) hiding overload behaviour
- **Coordinated omission** silently deleting the tail from your percentiles
- **A saturated load generator** measuring its own limits
- **Skipping the soak test**, and shipping the leak
- **Testing against third parties** without permission
- **Not warning on-call**, causing a self-inflicted incident response
- **Treating the breaking point as capacity** instead of the knee
- **Running production at 90% of measured capacity**
- **One-off testing**, with results that are stale a quarter later
- **Leaving test data in production**

---

## 20. Open Source Technologies

- **k6** — scripts in JavaScript, arrival-rate executors, good CI integration; the modern default
- **Locust** — scenarios in Python; excellent for complex user behaviour
- **Gatling** — Scala/Java, strong reporting, well-suited to large tests
- **wrk / wrk2** — minimal and extremely fast; `wrk2` corrects coordinated omission
- **vegeta** — simple constant-rate HTTP testing from the command line
- **Apache JMeter** — the veteran; capable, heavyweight, GUI-oriented
- **pgbench**, **sysbench** — load the database directly
- **Toxiproxy** — inject latency and failures into dependencies during a test
- **HdrHistogram** — honest latency distributions
- **Prometheus** + **Grafana** — the server-side half that makes results interpretable

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Derive your top five endpoints by volume from access logs. Is the ranking what you expected?
- [ ] Compare your staging data volume with production. If it is much smaller, your tests are misleading.
- [ ] Run a ramp test and find the latency knee. Compare it with your current peak traffic.
- [ ] Note what saturated first. Most teams are surprised.
- [ ] Run a spike test and time how long autoscaling takes to catch up.
- [ ] Run a six-hour soak test and watch memory.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
realistic scenarios → load generators (arrival rate) → production-like environment
                                  → client metrics + SERVER metrics → the knee
```

## 2. Request Flow

```text
Input       synthetic traffic shaped like real traffic, against real data volumes
    ↓
Processing  ramp until latency climbs, observing what saturates on the server side
    ↓
Output      a capacity number, the first bottleneck, and the shape of the failure
```

## 3. Real-World Usage

Load testing earns its keep in two places: before events with predictable traffic, and as a small regression gate in CI. The consistent lesson from real programmes is that the first bottleneck is almost never CPU — it is connection pools, thread pools, or a query whose plan changed with data volume. Which is exactly why the size of the test dataset matters more than the sophistication of the script.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Generating realistic synthetic load to find capacity and failure behaviour |
| **Why does it exist?** | Because systems fail abruptly, and users should not be the ones to discover it |
| **Where does it belong?** | Against a production-like environment, plus a small regression check in CI |
| **When should I use it?** | Before predictable traffic events, when sizing, and after architectural change |

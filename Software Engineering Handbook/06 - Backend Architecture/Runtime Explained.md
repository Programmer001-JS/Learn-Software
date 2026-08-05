# Runtime Explained — Backend Perspective

> [!NOTE]
> **This note is the backend half of a pair.**
> The language-level concept — what a runtime *is*, garbage collection, JIT compilation — lives in **[03 - Runtime Explained](../03%20-%20Programming%20Languages%20and%20Runtime/Runtime%20Explained.md)**. Read that first.
> This note covers the part that only matters once the runtime is serving traffic: how many processes to run, how concurrency actually behaves under load, and what it costs you in memory and cold starts.

> **In one line —** your runtime's concurrency model dictates your deployment topology; choosing worker counts without understanding it is guesswork.

| | |
|---|---|
| **Category** | Operational Concept |
| **Architectural Layer** | Server runtime |
| **Related notes** | [Runtime Explained (language)](../03%20-%20Programming%20Languages%20and%20Runtime/Runtime%20Explained.md) · [Web Servers](Web%20Servers.md) · [Gunicorn](Gunicorn.md) · [Uvicorn](Uvicorn.md) · [Process](../02%20-%20Computer%20Science%20Fundamentals/Process.md) · [Thread](../02%20-%20Computer%20Science%20Fundamentals/Thread.md) |

---

## 1. The question this note answers

*How many workers should I run, and why?*

Every backend deployment has to answer it, and the correct answer comes entirely from the runtime's concurrency model.

---

## 2. The three concurrency models

```text
PROCESS-BASED       CPython + Gunicorn, PHP-FPM
                    one request per worker process at a time (plus async inside)
                    parallelism = number of processes

THREAD-BASED        JVM, .NET
                    a thread pool; real parallelism inside one process
                    parallelism = number of cores

EVENT-LOOP          Node.js, Python asyncio, Go (hybrid)
                    one thread, thousands of concurrent waits
                    parallelism = 1 per loop; scale with processes
```

> [!IMPORTANT]
> **Concurrency is not parallelism.** An event loop handles 10,000 concurrent requests on one core — it is highly *concurrent* and not at all *parallel*. Confusing the two produces both under- and over-provisioned deployments.

---

## 3. Choosing worker counts

| Runtime | Typical topology | Reason |
|---|---|---|
| **Python (sync)** | `workers = 2 × cores + 1` | GIL: parallelism only via processes |
| **Python (async)** | `workers = cores`, high concurrency each | Event loop handles waiting; processes give cores |
| **Node.js** | `processes = cores` (cluster/PM2) | One loop per core |
| **JVM / .NET** | 1 process, thread pool sized by the runtime | Real threads, no GIL |
| **Go** | 1 process | The runtime schedules goroutines across all cores itself |

```text
Is the work CPU-bound or I/O-bound?
    ↓ I/O-bound                          ↓ CPU-bound
async / event loop wins             more processes or real threads
high concurrency per worker         parallelism = cores, no more
```

---

## 4. Memory is usually the binding constraint

```text
4 Gunicorn workers × 200 MB  =  800 MB before serving a single request
    ↓
Container limit 512 MB
    ↓
OOM killed — exit code 137
```

> [!CAUTION]
> Each worker process carries a full copy of the runtime and loaded libraries. In Python and Node this is often hundreds of megabytes. **Set container memory limits from measured worker memory × worker count**, not from a guess.

---

## 5. Cold starts

| Runtime | Cold start | Consequence |
|---|---|---|
| **Go** | ~1 ms | Ideal for serverless |
| **Node.js** | ~50 ms | Good for serverless |
| **Python** | ~50 ms + imports (often seconds) | Heavy imports dominate |
| **.NET** | ~100 ms, ~10 ms with AOT | Fine either way |
| **JVM** | 1–3 s, ~50 ms with GraalVM native | Poor for serverless without AOT |

This matters for [Lambda](../12%20-%20Cloud%20Architecture/Lambda.md) and for autoscaling: a runtime that takes three seconds to start cannot respond to a traffic spike quickly.

---

## 6. Graceful shutdown

```text
Deploy starts
    ↓
Orchestrator sends SIGTERM
    ↓
Runtime should: stop accepting new requests
                finish in-flight requests
                close database connections
                exit
    ↓
Grace period expires → SIGKILL → in-flight requests die
```

> [!CAUTION]
> A backend that ignores `SIGTERM` drops requests on every deployment. This is one of the most common causes of mysterious intermittent 502s during releases, and handling one signal fixes it.

---

## 7. Connection pooling

Every worker keeps its own database connection pool.

```text
4 workers × pool of 10  =  40 connections
16 replicas × 4 workers × 10  =  640 connections
    ↓
PostgreSQL default max_connections = 100
    ↓
"too many connections" under load
```

> [!TIP]
> Size pools by **total connections across all workers and all replicas**, and put a pooler such as PgBouncer in front when the number gets large.

---

## 8. Real World Example

- **Gunicorn + Uvicorn workers** is the standard Python production setup precisely because it combines process-level parallelism (working around the GIL) with an event loop inside each worker.
- **Node.js cluster mode** exists for the same reason: one loop cannot use eight cores.
- **JVM services** run as a single process with a large heap, because threads give real parallelism without extra processes.

---

## 9. Mental Model

> [!NOTE]
> **Workers are checkout tills; the concurrency model decides how many customers one till can serve at once.**
>
> A synchronous worker serves one customer and waits while they find their wallet. An async worker takes the next customer while the first searches. Threads let one till serve several customers genuinely simultaneously. Buying more tills helps only if the till was the bottleneck — not if everyone is waiting on the card machine.

---

## 10. Key Takeaway

> [!IMPORTANT]
> Worker count follows from the runtime's concurrency model and from whether your work is CPU- or I/O-bound — and memory per worker is usually what actually limits you.

---

## 11. Common Mistakes

- **Copying a worker count from a blog post** without knowing the model
- **Adding workers for an I/O-bound service** where async would serve far more with less memory
- **Ignoring memory per worker** until the container is OOM-killed
- **Not handling SIGTERM**, dropping requests on every deploy
- **Sizing connection pools per worker** instead of across the fleet
- **Using a slow-starting runtime for serverless**

---

## 12. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 13. Workbook Exercise

- [ ] Work out your total database connections: workers × pool size × replicas. Compare it with your database limit.
- [ ] Measure the memory of one worker under load and check it against your container limit.
- [ ] Verify your application shuts down gracefully on SIGTERM.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Load balancer
    ↓
N worker processes  (count set by the concurrency model)
    ↓
Runtime: threads or event loop
    ↓
Connection pool → database
```

## 2. Request Flow

```text
Input       concurrent requests
    ↓
Processing  distributed across workers; each handles them per its concurrency model
    ↓
Output      responses, bounded by CPU, memory or connection limits
```

## 3. Real-World Usage

The standard FastAPI production deployment — **Gunicorn managing Uvicorn workers** — exists entirely because of this note's subject: processes work around the GIL, and the event loop inside each one handles the waiting.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | How a runtime's concurrency model shapes backend deployment |
| **Why does it matter?** | Because worker counts, memory limits and pool sizes all follow from it |
| **Where does it belong?** | Between the web server and your application code |
| **When should I use it?** | Every time you size a deployment or debug capacity |

# BullMQ

> **In one line —** the Node.js job queue built on Redis: typed, well-designed, and it does not block your event loop as long as you keep the heavy work out of it.

| | |
|---|---|
| **Category** | Job Queue Library |
| **Architectural Layer** | Application |
| **Language** | TypeScript / Node.js |
| **Backed by** | Redis |
| **Related notes** | [Background Workers](Background%20Workers.md) · [Redis](../08%20-%20Databases%20and%20Data/Redis.md) · [Node.js Runtime](../03%20-%20Programming%20Languages%20and%20Runtime/Node.js%20Runtime.md) · [Event Loop](../05%20-%20Frontend%20Architecture/Event%20Loop.md) |

---

## 1. Short Definition

*What is it?*

BullMQ is a Node.js job queue library backed by Redis. It provides queues, workers, retries with backoff, scheduling, rate limiting, priorities and job dependencies — with full TypeScript typing.

---

## 2. Purpose

*What is its main purpose?*

To move slow work off the Node [event loop](../05%20-%20Frontend%20Architecture/Event%20Loop.md), where a single blocking operation would stall every concurrent request in the process.

---

## 3. Problem

*What engineering problem does it solve?*

```text
Node.js is single-threaded
    ↓
A 3-second image resize inside a request handler
    ↓
EVERY concurrent request in that process waits 3 seconds
    ↓
Health checks time out; the container is restarted
```

Moving that work to a separate worker process is not an optimisation in Node — it is a correctness requirement.

---

## 4. Architecture Position

```text
Express / NestJS API
    ↓  queue.add()
Redis  (queue storage)
    ↓
Worker PROCESSES — separate from the API
    ↓
Database · storage · external APIs
    ↓
Bull Board dashboard
```

> [!IMPORTANT]
> **Workers must run in a separate process from the API**, not in the same one. Running a worker inside your web server puts the CPU-heavy work back on the event loop you were trying to protect.

---

## 5. The basic shape

```typescript
// producer
await queue.add('resize', { imageId: 42 }, {
  attempts: 5,
  backoff: { type: 'exponential', delay: 1000 },
  removeOnComplete: 1000,       // ← keep Redis from growing forever
  removeOnFail: 5000,
});

// worker — a SEPARATE process
new Worker('images', async job => {
  const image = await db.image.findUnique({ where: { id: job.data.imageId } });
  await resize(image);
}, { concurrency: 5 });
```

> [!CAUTION]
> **`removeOnComplete` and `removeOnFail` are not optional.** By default BullMQ keeps completed and failed job records in Redis indefinitely. A busy queue will consume all available memory — this is the most common BullMQ production incident.

---

## 6. Features worth knowing

| Feature | Use |
|---|---|
| **Repeatable jobs** | Cron-style scheduling, stored in Redis |
| **Rate limiting** | "Maximum 100 jobs per minute" — respect third-party API limits |
| **Priorities** | Urgent jobs jump the queue |
| **Flows** | Parent/child job dependencies |
| **Delayed jobs** | "Run this in 24 hours" |
| **Concurrency** | Jobs processed in parallel per worker |

> [!TIP]
> Built-in **rate limiting** is genuinely useful and unusual in job libraries. When a third-party API allows 100 requests per minute, enforcing it at the queue is far more reliable than sprinkling sleeps through your code.

---

## 7. Concurrency versus blocking

```text
concurrency: 5
    ↓
Five jobs processed "at once" — on ONE event loop
    ↓
Genuinely concurrent for I/O-bound work (database, HTTP)
    ↓
NOT parallel for CPU-bound work — image processing still blocks
```

> [!CAUTION]
> Raising `concurrency` does not give you CPU parallelism. For CPU-heavy jobs you need **multiple worker processes**, or BullMQ's **sandboxed processors**, which run the job in a child process.

---

## 8. Real World Example

- **Node backends** doing email, image processing, webhooks and report generation.
- **NestJS** has first-class BullMQ integration via `@nestjs/bullmq`.
- **Rate-limited third-party synchronisation** — the queue enforces the limit centrally.
- **Bull Board** provides a web dashboard for inspecting, retrying and removing jobs.

---

## 9. Communication and Dependencies

- **Redis** — required, and it becomes a critical dependency
- **Separate worker processes** — deployed and scaled independently
- **Bull Board** or Taskforce for monitoring
- **TypeScript**, for the typing benefits it is designed around

---

## 10. Alternatives

```text
BullMQ         actively maintained, typed, feature-rich    ← the default
    ↓
Bull (v3)      the predecessor; use BullMQ for new work
    ↓
Agenda         MongoDB-backed
    ↓
pg-boss        PostgreSQL-backed — no Redis needed
    ↓
Graphile Worker  PostgreSQL, transactional enqueue
```

> [!TIP]
> **pg-boss and Graphile Worker are worth considering** if you already run PostgreSQL and do not want Redis as an additional critical dependency — and they let you enqueue inside the same transaction as your data change, which removes a real class of bug.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use BullMQ for Node background jobs where you want retries, scheduling and rate limiting, and Redis is already part of the stack.

> [!CAUTION]
> - **Not for very long retention** — job history lives in Redis, which is RAM.
> - **Not for durable multi-day workflows** — that is Temporal's domain.
> - **Not inside the API process** — that defeats the purpose entirely.

---

## 12. Advantages and Disadvantages

**Advantages**
- Excellent TypeScript support
- Rate limiting, priorities, flows and repeatable jobs built in
- Good dashboard tooling
- Fast — Redis-backed
- Actively maintained, with a clear upgrade path from Bull

**Disadvantages**
- Requires Redis, and depends on it entirely
- **Unbounded job retention by default** — a real memory hazard
- Concurrency does not give CPU parallelism
- Redis persistence is weaker than a disk-based broker
- Enqueue is not transactional with your database writes

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Enqueue** | ~1 ms |
| **Redis memory** | Grows with retained job records — trim aggressively |
| **Concurrency** | Helps I/O-bound work; use processes for CPU-bound |
| **Throughput** | Thousands of jobs per second |

---

## 14. Security Considerations

- **Redis must not be reachable from untrusted networks**, and must require authentication
- **Validate `job.data`** — it is untrusted input from a queue
- **Authenticate Bull Board** — it exposes job payloads and allows job manipulation
- **Do not put secrets or personal data in job payloads**; pass identifiers
- **Sandboxed processors run in child processes** — the same input validation applies
- **npm supply chain**: workers frequently run with database and third-party credentials, so their dependency tree deserves the same scrutiny as the API's

---

## 15. Mental Model

> [!NOTE]
> **BullMQ is a ticketing system for a workshop.**
>
> The front desk writes a ticket and returns to the counter immediately. Workers in the back take tickets, retry failed jobs, and respect a rule about how many can be worked per hour. The one thing you must remember is to throw away completed tickets — otherwise the workshop fills with paper.

---

## 16. Mini Architecture Diagram

```text
API process ──queue.add()──► Redis
                               ↓
        ┌── worker process ──┬── worker process ──┐
        │  concurrency: 5    │  concurrency: 5    │
        └─────────┬──────────┴─────────┬──────────┘
                  ↓                    ↓
            Database · storage · external APIs
                  ↓
            Bull Board (authenticated)
```

---

## 17. Complete Request Flow

```text
POST /images — file uploaded
    ↓
Stored; row created with status "pending"
    ↓
queue.add('resize', { imageId: 42 }, { attempts: 5, removeOnComplete: 1000 })
    ↓
201 returned immediately — the event loop is never blocked
    ↓
Worker process picks it up
    ↓
Loads image 42 fresh from the database
    ↓
Already resized? → return           ← idempotency
    ↓
Resize runs in a sandboxed processor, off the worker's event loop
    ↓
Status updated to "ready"
    ↓
Job record removed per removeOnComplete
    ↓
─────────── failure ───────────
Throws → exponential backoff → 1s, 2s, 4s, 8s, 16s
    ↓
Attempts exhausted → moved to failed → visible in Bull Board → alert
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> BullMQ keeps slow work off Node's single event loop — run workers as separate processes, and always set `removeOnComplete`, or Redis will fill with job history.

---

## 19. Common Mistakes

- **No `removeOnComplete` / `removeOnFail`** — unbounded Redis growth
- **Running workers inside the API process**
- **Expecting `concurrency` to give CPU parallelism**
- **Passing whole objects** in job data instead of IDs
- **Unauthenticated Bull Board**, exposing payloads and controls
- **No monitoring** of queue depth and failed-job count
- **Non-idempotent jobs**
- **Assuming enqueue is transactional** with your database write

---

## 20. Open Source Technologies

- **BullMQ**, **@nestjs/bullmq**
- **Bull Board**, **Taskforce.sh** — dashboards
- **pg-boss**, **Graphile Worker** — PostgreSQL-backed alternatives
- **Agenda** — MongoDB-backed
- **Temporal** — durable workflows, a different category

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Check whether your queues set `removeOnComplete`, and measure how much Redis memory job history currently uses.
- [ ] Confirm workers run in a separate process from your API.
- [ ] Add a CPU-heavy job and observe whether it blocks other jobs in the same worker.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
API → Redis queue → worker processes → services → dashboard
```

## 2. Request Flow

```text
Input       a job added with retry, backoff and retention options
    ↓
Processing  consumed by a separate worker process, retried on failure
    ↓
Output      completed work, with job records trimmed
```

## 3. Real-World Usage

**NestJS applications** use BullMQ as the standard background processing layer. Its built-in rate limiting is particularly valuable when synchronising with third-party APIs that enforce strict request limits.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A Redis-backed job queue library for Node.js |
| **Why does it exist?** | Because blocking Node's event loop stalls every concurrent request |
| **Where does it belong?** | Between an API process and separate worker processes |
| **When should I use it?** | Node background jobs, with Redis already in the stack |

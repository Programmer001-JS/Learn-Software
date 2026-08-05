# Background Workers

> **In one line —** separate processes that do the slow work a request should never wait for — and they need the same care as your API, which they rarely get.

| | |
|---|---|
| **Category** | Overview note *(hub for this sub-section)* |
| **Architectural Layer** | Application |
| **Sub-topics** | [Celery](Celery.md) · [BullMQ](BullMQ.md) · [Hangfire](Hangfire.md) |
| **Related notes** | [Message Queues](Message%20Queues.md) · [Process](../02%20-%20Computer%20Science%20Fundamentals/Process.md) · [Monitoring and Alerting](../14%20-%20Scalability%20and%20Reliability/Monitoring%20and%20Alerting.md) |

---

## 1. Short Definition

*What is it?*

A background worker is a long-running process that consumes tasks from a [queue](Message%20Queues.md) and executes them outside the request/response cycle.

---

## 2. Purpose

*What is its main purpose?*

To keep requests fast, to make slow work retryable, and to survive failures of third-party services without failing user actions.

---

## 3. What belongs in a worker

```text
✓ Email and notifications          slow, retryable, unreliable third party
✓ Image and video processing       CPU-heavy, seconds to minutes
✓ Report generation and exports    large queries, big outputs
✓ Third-party API synchronisation  they will be down sometimes
✓ Scheduled and recurring jobs     cleanup, billing runs, digests
✓ Bulk operations                  "email 50,000 users"

✗ Anything the user must see the result of immediately
✗ Work under 50 ms — the queue overhead exceeds the benefit
```

---

## 4. Architecture Position

```text
API  ──enqueue──►  QUEUE  ──►  Worker pool
                                   ├─ worker 1
                                   ├─ worker 2
                                   └─ worker 3
                                       ↓
                    Database · storage · external APIs
                                       ↓
                              Result / status store
```

> [!IMPORTANT]
> **Workers are deployed and scaled separately from the API.** They have different resource profiles (often CPU- or memory-heavy), different scaling triggers (queue depth, not request rate), and different failure behaviour.

---

## 5. The rules that make workers reliable

```text
1. IDEMPOTENT        redelivery is normal; running twice must be safe
2. SMALL PAYLOADS    pass an ID, not the whole object — the row may have changed
3. TIMEOUTS          every task needs one, or one hung job holds a worker forever
4. RETRIES + BACKOFF exponential, with a cap
5. DEAD-LETTER PATH  a permanently failing job must stop and alert
6. GRACEFUL SHUTDOWN finish the current task on SIGTERM before exiting
```

> [!CAUTION]
> **Pass identifiers, not serialised objects.** A task carrying a full user object executes against data that may be minutes old, and it silently breaks whenever you change the model's shape. Pass `user_id` and load it fresh.

---

## 6. Scheduled jobs

```text
Cron on one server        → dies with that server
Cron on every server      → the job runs N times          ← the classic bug
Scheduler + queue         → one schedule, one enqueue, workers execute
Kubernetes CronJob        → the orchestrator guarantees a single run
```

> [!TIP]
> If you must run cron on multiple instances, guard it with a [distributed lock](../08%20-%20Databases%20and%20Data/Distributed%20Lock.md) — and still make the job idempotent, because that lock is an efficiency measure rather than a guarantee.

---

## 7. Monitoring — the part that is usually missing

Workers fail silently. Nobody gets a 500; the work simply does not happen.

```text
✓ Queue depth              growing = consumers cannot keep up
✓ Oldest message age       the clearest signal of a real backlog
✓ Task duration (p95/p99)  a slow task starves the pool
✓ Failure rate and DLQ size
✓ Worker liveness          are they even running?
```

> [!IMPORTANT]
> **Alert on message age, not only on errors.** A queue with zero errors and a two-hour-old oldest message means everything is "working" and nothing is getting done — which is a far more common failure than crashes.

---

## 8. Real World Example

- **Email and notification pipelines** — the most common worker workload anywhere.
- **Video encoding** — enqueue on upload, process for minutes, notify on completion.
- **Nightly billing and reconciliation** — scheduled, idempotent, alert-on-failure.
- **Search index updates** — triggered by database changes, processed asynchronously.

---

## 9. Task granularity

```text
✗ ONE TASK: "email 50,000 users"
      one failure at user 30,000 → retry sends 30,000 duplicates

✓ FAN-OUT: one task per user
      each fails and retries independently; progress is visible;
      work distributes across all workers
```

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Move work to a worker when it is slow, retryable, or depends on something outside your control. Keep it in the request when the user needs the result now.

> [!CAUTION]
> Do not put trivial work in a queue. Enqueue, serialise, poll, deserialise and acknowledge costs more than a 10 ms function call — and you have added a failure mode for nothing.

---

## 11. Advantages and Disadvantages

**Advantages**
- Fast responses regardless of how slow the work is
- Automatic retries with backoff
- Independent scaling from the API
- Failures isolated from user requests
- Absorbs traffic spikes

**Disadvantages**
- Eventual consistency — the work is not done when the response returns
- Debugging spans producer, queue and worker
- Silent failure unless monitored properly
- A second deployment target with its own resource profile
- Idempotency required everywhere

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Request latency** | Improves substantially — slow work removed from the path |
| **Worker memory** | Often the constraint; image and ML tasks are heavy |
| **Concurrency** | Match to the workload: CPU-bound → cores; I/O-bound → higher |
| **Queue depth** | The primary scaling signal |

---

## 13. Security Considerations

> [!CAUTION]
> **Workers routinely run with broader permissions than the API** — database writes, file system access, third-party credentials — and are frequently excluded from security review because they are "internal". They are not internal: their input comes from a queue, and a queue is a trust boundary.

- **Validate task payloads** exactly as you would an HTTP request
- **Never deserialise untrusted data** — `pickle` in Celery is remote code execution; use JSON
- **Least privilege per worker** — a thumbnail worker does not need billing credentials
- **Do not log full payloads** — they often contain personal data
- **Authorisation belongs in the [service layer](../07%20-%20Backend%20Design%20Patterns/Services.md)**, so a worker calling the same code cannot bypass a rule the API enforces
- **Rate-limit enqueueing** — an unbounded queue is a resource-exhaustion vector

---

## 14. Mental Model

> [!NOTE]
> **Background workers are the back-of-house kitchen.**
>
> The waiter takes the order and returns to the floor immediately. The kitchen works through the rail at its own pace. If the kitchen is understaffed nobody at the tables sees an error — the food simply never arrives, which is exactly why you watch the rail rather than waiting for complaints.

---

## 15. Mini Architecture Diagram

```text
API  ──enqueue(id)──►  Queue
                          ↓
                   Worker pool (scaled on queue depth)
                          ↓
              load fresh data by ID → do the work
                          ↓
        success → ack        failure → backoff → retry → DLQ → alert
```

---

## 16. Complete Request Flow

```text
User uploads a video
    ↓
API stores the file, creates a row with status = "pending"
    ↓
Enqueues { video_id: 42 }  — an ID, not the file
    ↓
201 returned in ~50 ms
    ↓
Worker picks up the task
    ↓
Loads video 42 fresh from the database
    ↓
Already processed? → acknowledge and stop            ← idempotency
    ↓
Encoding runs for 4 minutes, with a timeout set
    ↓
Status updated to "ready"; user notified
    ↓
─────────── failure ───────────
Encoder crashes → no ack → redelivered
    ↓
Retries with backoff: 1s, 2s, 4s, 8s, 16s
    ↓
Still failing → dead-letter queue → alert → investigated
    ↓
Meanwhile the API and every other user are unaffected
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> Workers make slow work retryable and keep requests fast — pass IDs rather than objects, make every task idempotent, and alert on message age rather than only on errors.

---

## 18. Common Mistakes

- **Passing whole objects** instead of identifiers
- **Non-idempotent tasks**
- **No timeout**, so one hung job occupies a worker indefinitely
- **No dead-letter queue or alerting** — silent failure
- **Monitoring errors but not queue depth or message age**
- **Cron on every instance**, running scheduled jobs N times
- **One enormous task** instead of fan-out
- **No graceful shutdown**, killing work on every deployment

---

## 19. Open Source Technologies

- **Celery**, **RQ**, **Dramatiq**, **arq** — Python
- **BullMQ**, **Bee-Queue** — Node
- **Sidekiq** — Ruby
- **Hangfire** — .NET
- **Temporal**, **Airflow** — durable workflows and orchestration
- **Flower**, **Bull Board**, **Hangfire Dashboard** — worker observability

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Check whether any task in your system passes a whole object rather than an ID.
- [ ] Verify you have an alert on oldest-message-age, not only on task errors.
- [ ] Deploy while a task is running and confirm it finishes rather than being killed.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
API → queue → worker pool → database / external services → status store
```

## 2. Request Flow

```text
Input       a task referencing an entity by ID
    ↓
Processing  executed by a worker, idempotently, with timeout and retries
    ↓
Output      completed work, or a dead-lettered task and an alert
```

## 3. Real-World Usage

**Video platforms** enqueue encoding on upload and process for minutes. The user gets an immediate response and a notification later — an interaction pattern that only works because the work is durable, retryable and monitored.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Separate processes executing queued work outside the request cycle |
| **Why does it exist?** | So slow, unreliable work does not block or fail user requests |
| **Where does it belong?** | Behind a queue, deployed and scaled independently |
| **When should I use it?** | Slow, retryable, or externally dependent work — not trivial tasks |

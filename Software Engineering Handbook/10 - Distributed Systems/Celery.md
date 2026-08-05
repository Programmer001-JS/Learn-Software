# Celery

> **In one line —** Python's standard background task framework: decorate a function, call it with `.delay()`, and a worker somewhere runs it.

| | |
|---|---|
| **Category** | Task Queue Framework |
| **Architectural Layer** | Application |
| **Language** | Python |
| **Brokers** | RabbitMQ, Redis, SQS |
| **Related notes** | [Background Workers](Background%20Workers.md) · [RabbitMQ](RabbitMQ.md) · [Redis](../08%20-%20Databases%20and%20Data/Redis.md) · [Django](../06%20-%20Backend%20Architecture/Django.md) · [CPython](../03%20-%20Programming%20Languages%20and%20Runtime/CPython.md) |

---

## 1. Short Definition

*What is it?*

Celery is a distributed task queue for Python. It turns ordinary functions into tasks that can be executed asynchronously by worker processes, with retries, scheduling and result tracking.

---

## 2. Purpose

*What is its main purpose?*

To make asynchronous execution feel like a normal function call, while handling serialisation, delivery, retries and worker management underneath.

---

## 3. Problem

*What engineering problem does it solve?*

Python's [GIL](../03%20-%20Programming%20Languages%20and%20Runtime/CPython.md) means CPU-bound work cannot be parallelised with threads, and slow I/O should not block a request. Celery provides separate worker **processes** consuming from a broker.

---

## 4. Architecture Position

```text
Django / FastAPI
    ↓  task.delay(id)
Broker  (RabbitMQ or Redis)
    ↓
Celery worker processes
    ↓
Result backend  (Redis / database) — optional
    ↓
Celery Beat → scheduled tasks
```

---

## 5. The basic shape

```python
@app.task(bind=True, max_retries=3, autoretry_for=(RequestException,),
          retry_backoff=True, acks_late=True)
def send_welcome_email(self, user_id):        # ← an ID, not a User object
    user = User.objects.get(id=user_id)
    mail.send(user.email, template="welcome")

send_welcome_email.delay(user.id)             # returns immediately
```

Every option in that decorator matters, and the defaults are not the ones you want — see below.

---

## 6. The defaults you must change

> [!CAUTION]
> Celery's out-of-the-box configuration is convenient rather than safe. Three settings account for most production surprises.

```text
acks_late = False  (default)
    → the task is acknowledged when RECEIVED, not when finished
    → a worker crash loses the task silently
    ✓ set acks_late = True

task_serializer = "json"
    → the pickle serializer is REMOTE CODE EXECUTION if the broker is reachable
    ✓ keep JSON; never enable pickle

worker_prefetch_multiplier = 4  (default)
    → one worker grabs four tasks; long tasks starve other workers
    ✓ set to 1 for long-running tasks
```

---

## 7. Retries

```python
@app.task(bind=True, max_retries=5, retry_backoff=True, retry_jitter=True)
def sync_crm(self, order_id):
    try:
        crm.push(order_id)
    except TemporaryError as exc:
        raise self.retry(exc=exc)     # 1s → 2s → 4s → 8s → 16s
```

> [!TIP]
> `retry_backoff` with `retry_jitter` is the correct default. Without jitter, a downstream outage causes every failed task to retry at exactly the same moment — a synchronised stampede against a service that is already struggling.

---

## 8. Workflows

```text
chain(a.s(), b.s(), c.s())      run in sequence, passing results along
group(a.s(), b.s(), c.s())      run in parallel
chord(group(...), callback.s()) run in parallel, then a callback
```

Useful for real pipelines — download, transform, notify — though deep chains become hard to debug. For genuinely complex workflows, **Temporal** or **Airflow** are better suited.

---

## 9. Celery Beat — scheduling

```python
app.conf.beat_schedule = {
    "nightly-cleanup": {"task": "tasks.cleanup", "schedule": crontab(hour=3)},
}
```

> [!CAUTION]
> **Run exactly one Beat instance.** Two schedulers means every job is enqueued twice. Use a single deployment replica, or a locking scheduler such as `celery-redbeat`.

---

## 10. Real World Example

- **Django plus Celery plus Redis** is the standard Python background stack.
- **Instagram** ran Celery at very large scale in its earlier architecture.
- **Typical workloads**: emails, PDF generation, image processing, third-party synchronisation, nightly reports.

---

## 11. Communication and Dependencies

- **A broker** — RabbitMQ (recommended for reliability) or Redis (simpler)
- **A result backend**, only if you actually need results
- **Worker processes**, deployed separately from the API
- **Flower** or Prometheus for monitoring

> [!TIP]
> **Disable the result backend unless you use it.** Storing a result for every task is a common and unnoticed source of Redis memory growth.

---

## 12. Alternatives

```text
Celery      the most features, the most configuration, the most footguns
    ↓
RQ          much simpler, Redis only, fewer capabilities
    ↓
Dramatiq    modern, saner defaults, smaller ecosystem
    ↓
arq         asyncio-native, lightweight
    ↓
Temporal    durable workflows with real state — a different category
```

> [!TIP]
> For a straightforward job queue, **RQ or Dramatiq are easier to run correctly**. Celery earns its complexity when you need routing, scheduling, workflows and multiple brokers.

---

## 13. When To Use / When NOT To Use

> [!TIP]
> Use Celery for Python background processing where you need retries, scheduling and routing — especially in a Django project, where the integration is well established.

> [!CAUTION]
> - **Not for sub-50 ms work** — the overhead exceeds the benefit.
> - **Not for long durable workflows** — a three-day approval process belongs in Temporal, not a task chain.
> - **Not in a new async codebase** without care — Celery's async support is limited; `arq` or `Dramatiq` fit better.

---

## 14. Advantages and Disadvantages

**Advantages**
- Mature, with a very large ecosystem
- Retries, scheduling, routing and workflows built in
- Multiple broker options
- Excellent Django integration
- Good monitoring tools

**Disadvantages**
- **Unsafe defaults** — `acks_late`, prefetch and serialisation all need changing
- Complex configuration surface
- Each worker carries a full Python interpreter — memory adds up
- Debugging distributed task failures is genuinely hard
- Documentation is large and occasionally contradictory

---

## 15. Performance Impact

| Aspect | Impact |
|---|---|
| **Overhead** | A few milliseconds per task |
| **Memory** | 100–300 MB per worker process — the usual constraint |
| **Concurrency** | Prefork for CPU-bound, gevent/eventlet for I/O-bound |
| **Result backend** | Storing every result grows unbounded without expiry |

---

## 16. Security Considerations

> [!CAUTION]
> **`task_serializer = "pickle"` is remote code execution.** Anyone who can enqueue a message to the broker can run arbitrary code on every worker. Celery defaults to JSON now, but older configurations and tutorials still enable pickle.

- **Never expose the broker** to untrusted networks; authenticate it
- **Validate task arguments** — they arrive from a queue, which is a trust boundary
- **Least privilege per worker queue** — route sensitive tasks to workers with the credentials, and nothing else
- **Do not log task arguments** wholesale; they frequently contain personal data
- **Flower must be authenticated** — it exposes task arguments and allows task control

---

## 17. Mental Model

> [!NOTE]
> **Celery is a dispatch office for a fleet of vans.**
>
> You hand over a job docket; the office decides which van takes it, retries if the delivery fails, and runs standing orders on a schedule. Powerful — and it comes with a thick manual whose default settings assume everything goes well.

---

## 18. Mini Architecture Diagram

```text
Django/FastAPI ──.delay(id)──► Broker (RabbitMQ)
                                    ↓
                      ┌── worker ──┬── worker ──┐
                      │ prefork    │ prefork    │
                      └─────┬──────┴─────┬──────┘
                            ↓            ↓
                    Database · external APIs
                            ↓
                   Result backend (optional, expiring)
       Celery Beat ──schedule──► Broker
```

---

## 19. Complete Request Flow

```text
POST /register
    ↓
User created; send_welcome_email.delay(user.id)  ← ID only
    ↓
Task serialised as JSON and published to the broker
    ↓
201 returned in ~30 ms
    ↓
Worker receives the task (prefetch = 1)
    ↓
Loads the user fresh from the database
    ↓
Sends the email
    ↓
acks_late=True → acknowledged only NOW, after success
    ↓
─────────── failure ───────────
SMTP unavailable → autoretry with backoff and jitter
    ↓
Exhausted after 5 attempts → dead letter → alert
    ↓
─────────── crash ───────────
Worker killed mid-task → never acknowledged → REDELIVERED
    ↓
Which is why the task must be idempotent
```

---

## 20. Key Takeaway

> [!IMPORTANT]
> Celery makes async execution look like a function call — but you must set `acks_late=True`, keep JSON serialisation, and tune prefetch, because the defaults lose tasks quietly.

---

## 21. Common Mistakes

- **Leaving `acks_late=False`** — silent task loss on worker crash
- **Enabling the pickle serializer**
- **Default prefetch** with long tasks, starving other workers
- **Passing model objects** instead of IDs
- **Result backend enabled and never cleaned**, growing forever
- **Multiple Beat instances**, double-scheduling every job
- **No monitoring** of queue depth and task age
- **Non-idempotent tasks**

---

## 22. Open Source Technologies

- **Celery**, **Celery Beat**, **celery-redbeat**
- **Flower** — the monitoring dashboard (authenticate it)
- **RQ**, **Dramatiq**, **arq** — simpler alternatives
- **RabbitMQ**, **Redis** — brokers
- **Temporal**, **Airflow** — durable workflows

---

## 23. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 24. Workbook Exercise

- [ ] Check your Celery configuration for `acks_late`, `task_serializer` and `worker_prefetch_multiplier`.
- [ ] Kill a worker mid-task and confirm the task is redelivered rather than lost.
- [ ] Check whether your result backend is enabled, used, and expiring.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
App → .delay() → broker → worker processes → database / APIs → optional results
```

## 2. Request Flow

```text
Input       a task name and JSON-serialisable arguments
    ↓
Processing  routed via a broker to a worker process, retried with backoff
    ↓
Output      completed work, acknowledged only after success
```

## 3. Real-World Usage

**Django plus Celery plus Redis** is the default Python background stack, and its most common production incident is task loss from the default `acks_late=False`. Knowing that one setting separates a working deployment from a quietly lossy one.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Python's standard distributed task queue framework |
| **Why does it exist?** | Because the GIL means background work needs separate processes |
| **Where does it belong?** | Between a Python application and a broker |
| **When should I use it?** | Python background jobs needing retries, scheduling and routing |

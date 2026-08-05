# Message Queues

> **In one line —** a buffer between a producer and a consumer, so slow work does not block the request that asked for it, and a crash does not lose the job.

| | |
|---|---|
| **Category** | Overview note *(hub for this sub-section)* |
| **Architectural Layer** | Integration |
| **Sub-topics** | [RabbitMQ](RabbitMQ.md) · [Kafka](Kafka.md) · [Redis Streams](Redis%20Streams.md) · [AWS SQS](AWS%20SQS.md) |
| **Related notes** | [Background Workers](Background%20Workers.md) · [Pub Sub](../08%20-%20Databases%20and%20Data/Pub%20Sub.md) · [Event Driven Architecture](Event%20Driven%20Architecture.md) |

---

## 1. Short Definition

*What is it?*

A message queue stores messages produced by one part of a system until another part is ready to process them. Each message is delivered to **exactly one** consumer.

---

## 2. Purpose

*What is its main purpose?*

To decouple **when** work is requested from **when** it is done — and to make that work survive a crash, a restart or a deployment.

---

## 3. Problem

*What engineering problem does it solve?*

```text
SYNCHRONOUS                           QUEUED
POST /register                        POST /register
    ↓                                     ↓
create user (20 ms)                   create user (20 ms)
send welcome email (2 s)              enqueue "send email"  (1 ms)
generate avatar (3 s)                 enqueue "make avatar" (1 ms)
call the CRM (1 s, sometimes down)    enqueue "sync CRM"    (1 ms)
    ↓                                     ↓
6 seconds; the CRM being down         ~25 ms; workers process it
fails the registration                 asynchronously and retry on failure
```

---

## 4. Architecture Position

```text
API (producer)
    ↓  enqueue
┌──────── QUEUE ────────┐
│  durable storage      │
│  ordering             │
│  retries              │
│  dead-letter queue    │
└───────────┬───────────┘
            ↓  one consumer per message
    Worker 1 · Worker 2 · Worker 3
            ↓
        Database · external APIs
```

---

## 5. Queue vs Pub/Sub vs Stream

> [!IMPORTANT]
> These three are constantly confused, and choosing the wrong one produces either lost work or duplicated work.

| | **Queue** | **[Pub/Sub](../08%20-%20Databases%20and%20Data/Pub%20Sub.md)** | **[Stream](Kafka.md)** |
|---|---|---|---|
| **Delivery** | **One** consumer | **Every** subscriber | Every consumer group |
| **Persistence** | Until acknowledged | Usually none | Retained for days |
| **Offline consumer** | Gets it later | **Misses it** | Reads from its offset |
| **Replay** | No | No | **Yes** |
| **Purpose** | Distribute work | Broadcast | Event log |

```text
"Send this email"              → QUEUE   (exactly one worker should send it)
"A user just signed up"        → PUB/SUB or STREAM (many may care)
"Every order event, replayable"→ STREAM
```

---

## 6. Delivery guarantees

```text
AT MOST ONCE    may be lost, never duplicated       — rarely acceptable
AT LEAST ONCE   never lost, MAY BE DUPLICATED       — the practical default
EXACTLY ONCE    the marketing claim
```

> [!CAUTION]
> **"Exactly once" delivery does not exist end to end.** A consumer can process a message and crash before acknowledging it; the broker cannot distinguish that from a consumer that never processed it, so it redelivers.
>
> The workable version is **at-least-once delivery plus idempotent consumers** — which gives you exactly-once *effects*. Design for this from the start.

---

## 7. Idempotency — the non-negotiable requirement

```text
✗ charge_card(order)                     retried → charged twice
✓ charge_card(order, idempotency_key)    retried → returns the first result

✗ balance += amount                      retried → double credit
✓ INSERT ... ON CONFLICT DO NOTHING      retried → no effect
```

> [!IMPORTANT]
> Every message consumer must be safe to run twice with the same input. This is not defensive over-engineering — redelivery is normal operation, not an error case.

---

## 8. Retries and dead letters

```text
Message fails
    ↓
Retry with EXPONENTIAL BACKOFF: 1s → 2s → 4s → 8s
    ↓
Still failing after N attempts
    ↓
DEAD LETTER QUEUE  ← inspect, fix, replay
```

> [!CAUTION]
> Without a dead-letter queue, a permanently failing message ("poison message") is retried forever, consuming a worker indefinitely and often filling the logs. A DLQ turns an infinite loop into an alert.

---

## 9. Choosing a broker

| | Best for | Note |
|---|---|---|
| **[RabbitMQ](RabbitMQ.md)** | Complex routing, per-message acknowledgement | The classic task queue |
| **[Kafka](Kafka.md)** | High throughput, replay, event log | Not really a task queue |
| **[Redis Streams](Redis%20Streams.md)** | You already run Redis | Persistent, consumer groups |
| **[AWS SQS](AWS%20SQS.md)** | Managed, no operations | Simple, effectively unlimited scale |
| **PostgreSQL `SKIP LOCKED`** | Modest volume, no new system | Genuinely underrated |

> [!TIP]
> **You may not need a broker at all.** `SELECT ... FOR UPDATE SKIP LOCKED` gives correct, transactional work distribution in a database you already run — and the job is enqueued in the same transaction as the data change, which removes an entire class of consistency bug.

---

## 10. Real World Example

- **Email, notifications, PDF generation, image processing, report exports** — the standard queue workloads.
- **Payment processing** — enqueued so a downstream outage delays rather than fails an order.
- **The outbox pattern** — write the message to a database table inside the same transaction as the business change, then relay it to the broker. This is how you avoid "the order was saved but the event was never published".

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Queue anything slow, anything that calls an unreliable third party, anything that can be retried, and anything the user does not need to wait for.

> [!CAUTION]
> - **Not for work whose result the user needs immediately** — that is a synchronous call.
> - **Not when strict ordering across all messages matters** — queues parallelise, which reorders. Partition by key if order matters within a key.
> - **Not as a database** — a queue is a pipe, not storage.

---

## 12. Advantages and Disadvantages

**Advantages**
- Fast responses; slow work moves off the request path
- Absorbs traffic spikes — the queue grows instead of the system failing
- Retries and failure isolation come free
- Producers and consumers scale independently
- Work survives restarts and deployments

**Disadvantages**
- Another system to run, monitor and secure
- Eventual consistency — the work is not done when the response returns
- Debugging spans producer, broker and consumer
- Requires idempotency everywhere
- Queue depth becomes a metric you must watch

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Producer latency** | ~1 ms to enqueue |
| **End-to-end** | Depends entirely on consumer capacity |
| **Queue depth** | **The key metric** — growing depth means consumers cannot keep up |
| **Scaling** | Add consumers; ordering guarantees weaken as you do |

> [!TIP]
> Alert on **queue depth and message age**, not just on errors. A silently growing backlog is the earliest signal that something downstream has slowed.

---

## 14. Security Considerations

> [!CAUTION]
> **Messages are data from another system, and must be validated exactly like an HTTP request.** A compromised producer, or anyone who can reach the broker, can enqueue anything.

- **Authenticate producers and consumers**; never leave a broker open on the network
- **Validate every message payload** against a schema before acting on it
- **Do not put sensitive data in messages** — publish identifiers and let the consumer fetch what it is authorised to see
- **Encrypt in transit**, and at rest for durable queues
- **A dead-letter queue accumulates real payloads** — it is a data store with the same sensitivity, and it is usually the least protected part of the system
- **Rate-limit producers** — an unbounded queue is a resource-exhaustion vector

---

## 15. Mental Model

> [!NOTE]
> **A message queue is a restaurant order rail.**
>
> The waiter clips the order to the rail and returns to the floor immediately — they do not stand in the kitchen waiting. Chefs take orders one at a time, and each order goes to exactly one chef. If the kitchen falls behind, the rail gets longer; nobody loses an order, and the dining room keeps working.

---

## 16. Mini Architecture Diagram

```text
API  ──enqueue──►  QUEUE  ──►  Worker 1
                     │     ──►  Worker 2
                     │     ──►  Worker 3
                     ↓
              Dead-letter queue  ← after N failed attempts
```

---

## 17. Complete Request Flow

```text
POST /orders
    ↓
Order written to the database
    ↓
Message written to the OUTBOX table — same transaction
    ↓
COMMIT  ← the order and the intent to publish are now atomic
    ↓
201 returned to the user (~25 ms)
    ↓
Relay process reads the outbox and publishes to the queue
    ↓
Worker receives the message
    ↓
Idempotency key checked → already processed? → acknowledge and stop
    ↓
Work performed: email sent, CRM updated
    ↓
ACKNOWLEDGED → removed from the queue
    ↓
Failure instead → backoff retry → after N attempts → dead-letter queue → alert
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> A queue delivers each message to one consumer and guarantees at-least-once delivery — so every consumer must be idempotent, and every queue needs a dead-letter path.

---

## 19. Common Mistakes

- **Non-idempotent consumers** — duplicates cause double charges and double emails
- **No dead-letter queue** — a poison message retried forever
- **Expecting exactly-once delivery**
- **Publishing to the queue before the transaction commits** — announcing work for data that was rolled back
- **No monitoring of queue depth or message age**
- **Sensitive payloads in messages**
- **Adding a broker** where `SKIP LOCKED` on the existing database would do
- **Assuming ordering** while running multiple consumers

---

## 20. Open Source Technologies

- **RabbitMQ**, **Kafka**, **NATS**, **Redis Streams** — brokers
- **AWS SQS**, **Google Pub/Sub**, **Azure Service Bus** — managed
- **Celery**, **BullMQ**, **Sidekiq**, **Hangfire** — worker frameworks
- **PostgreSQL `SKIP LOCKED`**, **pg-boss**, **Solid Queue** — database-backed queues
- **Debezium** — change data capture, an alternative to the outbox pattern

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Take one queue consumer and work out what happens if the same message arrives twice.
- [ ] Check whether every queue in your system has a dead-letter queue and an alert on it.
- [ ] Find one place that publishes a message before the database transaction commits.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Producer → queue (durable) → one consumer → dead-letter queue on repeated failure
```

## 2. Request Flow

```text
Input       a task enqueued by a producer
    ↓
Processing  delivered at least once to exactly one consumer, retried with backoff
    ↓
Output      work completed idempotently, or parked in a dead-letter queue
```

## 3. Real-World Usage

**The outbox pattern** is how mature systems avoid the most common queue bug: writing the message to a database table in the same transaction as the business change, then relaying it. Without it, a crash between commit and publish loses the event silently.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A durable buffer delivering each message to exactly one consumer |
| **Why does it exist?** | So slow or unreliable work does not block or fail a request |
| **Where does it belong?** | Between a producer and background workers |
| **When should I use it?** | For anything slow, retryable, or dependent on an external service |

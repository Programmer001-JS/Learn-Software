# Kafka

> **In one line —** a durable, replayable log of events rather than a queue: messages are not deleted when read, so every consumer reads at its own pace and can rewind.

| | |
|---|---|
| **Full name** | Apache Kafka |
| **Category** | Event Streaming Platform |
| **Architectural Layer** | Integration / Data |
| **Origin** | LinkedIn, open-sourced 2011 |
| **Related notes** | [Message Queues](Message%20Queues.md) · [RabbitMQ](RabbitMQ.md) · [Event Driven Architecture](Event%20Driven%20Architecture.md) · [Redis Streams](Redis%20Streams.md) |

---

## 1. Short Definition

*What is it?*

Kafka is a distributed, append-only **log**. Producers append events to topics; consumers read from a position they control. Nothing is deleted on read — data is retained for a configured period.

---

## 2. Purpose

*What is its main purpose?*

To be the durable, replayable record of everything that happened in a system, readable by many independent consumers at very high throughput.

---

## 3. Problem

*What engineering problem does it solve?*

```text
QUEUE                                 LOG (Kafka)
message consumed → deleted            event appended → retained 7 days
    ↓                                     ↓
only one consumer ever sees it        analytics, search indexing, billing and
    ↓                                   ML pipelines all read the SAME events
a new consumer added later                ↓
sees nothing that came before         a new consumer added today can replay
                                        the last week from the beginning
```

> [!IMPORTANT]
> **Replay is the defining feature.** Deploy a bug in a consumer, fix it, reset the offset, and reprocess. No other messaging system in common use makes that routine.

---

## 4. Architecture Position

```text
Producers
    ↓
┌──────────── TOPIC ────────────┐
│  Partition 0: [0][1][2][3]... │  ← ordered within a partition
│  Partition 1: [0][1][2]...    │
│  Partition 2: [0][1][2][3]... │
└───────────────┬────────────────┘
                ↓
    Consumer group A          Consumer group B
    (each partition to one    (reads the same events
     consumer in the group)     independently)
```

---

## 5. Partitions — where the model lives

```text
A topic is split into partitions
    ↓
Ordering is guaranteed WITHIN a partition, never across them
    ↓
Messages with the same KEY always go to the same partition
    ↓
    key = order_id  →  all events for one order stay in order
    ↓
Parallelism ceiling = number of partitions
```

> [!IMPORTANT]
> Two consequences follow immediately, and both surprise people:
> - **Global ordering does not exist.** If you need it, you need one partition — and therefore one consumer.
> - **You cannot have more consumers in a group than partitions.** Extra consumers sit idle. Partition count is a capacity decision made early and awkward to change later.

---

## 6. Consumer groups and offsets

```text
Each consumer group tracks its own OFFSET per partition
    ↓
Group "billing"    at offset 15,000
Group "analytics"  at offset 12,300
Group "search"     at offset 15,000
    ↓
All reading the same topic, independently, at different positions
```

Committing an offset is what marks progress. Committing **before** processing gives at-most-once delivery (loss); committing **after** gives at-least-once (duplicates) — which is why consumers must be idempotent.

---

## 7. Retention

```text
Time-based     retain 7 days                — the common default
Size-based     retain 100 GB per partition
Compacted      keep the LATEST value per key, forever
```

> [!TIP]
> **Log compaction** turns a topic into a changelog: the current state of every key, retained indefinitely. It is how Kafka can act as a source of truth for things like configuration or entity state, rather than only a transport.

---

## 8. Real World Example

- **LinkedIn, Uber, Netflix** — trillions of events per day; Kafka was built for exactly this.
- **Change data capture** — Debezium streams database changes into Kafka, so other systems react without polling.
- **Event sourcing** — the log *is* the system of record; state is derived by replaying it.
- **Metrics and log aggregation** pipelines.

---

## 9. Kafka vs RabbitMQ

| | Kafka | [RabbitMQ](RabbitMQ.md) |
|---|---|---|
| **Model** | Retained log | Queue, deleted on ack |
| **Replay** | **Yes** | No |
| **Throughput** | Millions/sec | Tens of thousands/sec |
| **Routing** | Topic and partition only | Rich exchange routing |
| **Per-message ack** | No — offset commits | **Yes** |
| **Operational cost** | **High** | Moderate |
| **Best for** | Event streaming, analytics | Task distribution |

```text
"Send this welcome email"   → RabbitMQ or SQS
"Record that a user signed up, for everyone who cares, replayable"  → Kafka
```

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Use Kafka when you need **replay**, **very high throughput**, or **many independent consumers of the same event stream** — analytics pipelines, event sourcing, change data capture.

> [!CAUTION]
> **Kafka is not a task queue and it is not a beginner's default.** It is a distributed system in its own right: partitions, replication, consumer group rebalancing, offset management and retention tuning are all yours to operate.
>
> For "send this email in the background", Kafka is substantially the wrong tool — use [RabbitMQ](RabbitMQ.md), [SQS](AWS%20SQS.md), or your database. Teams routinely adopt Kafka for workloads a single Postgres table would have handled.

---

## 11. Advantages and Disadvantages

**Advantages**
- Extremely high throughput
- Durable and replayable
- Many independent consumer groups on one stream
- Horizontal scaling through partitions
- The de facto standard for event streaming, with a large ecosystem

**Disadvantages**
- **Significant operational complexity**
- No per-message acknowledgement or selective retry
- No rich routing
- Partition count is hard to change after the fact
- Consumer group rebalancing pauses processing
- Substantial infrastructure cost for modest workloads

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Throughput** | Millions of messages/second on a modest cluster |
| **Latency** | Single-digit milliseconds; batching trades latency for throughput |
| **Storage** | Retention × volume — plan disk deliberately |
| **Rebalancing** | Consumers pause while a group rebalances |
| **Partitions** | Sets the parallelism ceiling |

> [!TIP]
> Kafka's speed comes from **sequential disk writes and zero-copy transfer**. Appending to a log is one of the fastest things a disk does — which is why a disk-backed system outperforms many in-memory ones here.

---

## 13. Security Considerations

- **Enable authentication (SASL) and TLS** — an unsecured Kafka cluster exposes every event in the business
- **ACLs per topic** — a service should read only the topics it needs
- **Retained events are a data store**: seven days of user events is personal data subject to the same rules as a database, including deletion requests
- **Validate messages** — use a schema registry so consumers reject malformed or unexpected payloads
- **Publish identifiers, not sensitive fields**, where consumers can fetch what they are authorised to see
- **Encryption at rest** for the brokers' disks

> [!CAUTION]
> **GDPR deletion and an immutable log are in direct tension.** Deleting one person's events from a retained, replicated, compacted log is genuinely hard. The usual answer is to store only identifiers in Kafka and keep the personal data in a system that supports deletion — decide this before you build, not after.

---

## 14. Mental Model

> [!NOTE]
> **Kafka is a newspaper archive; RabbitMQ is a to-do list.**
>
> Items on a to-do list are crossed off and gone. The archive keeps every issue: different readers browse different years at their own pace, and a new reader can start from any date. Nothing is removed because someone read it.

---

## 15. Mini Architecture Diagram

```text
Producers (key = order_id)
    ↓
Topic "orders"
 ├ Partition 0 ──► consumer A1 ┐
 ├ Partition 1 ──► consumer A2 ├ group "billing"     offset 15,000
 └ Partition 2 ──► consumer A3 ┘
    │
    └──────────► group "analytics"  offset 12,300
    └──────────► group "search"     offset 15,000
```

---

## 16. Complete Request Flow

```text
Order created
    ↓
Producer appends to topic "orders" with key = order_id
    ↓
Partition chosen by hash(key) → all events for this order stay ordered
    ↓
Written to the leader, replicated to followers
    ↓
acks=all → acknowledged only once replicas have it
    ↓
Retained for 7 days, regardless of who reads it
    ↓
Billing group reads and commits its offset
Analytics group reads the same events, hours later
Search group reads them in real time
    ↓
Billing consumer had a bug → fix it → RESET THE OFFSET → reprocess
    ↓
Nothing else in the system is affected
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> Kafka is a retained, replayable log rather than a queue — its power is that many consumers read the same events independently, and its cost is that it is a distributed system you must operate.

---

## 18. Common Mistakes

- **Using it as a task queue** where RabbitMQ or SQS would be far simpler
- **Adopting it for scale you do not have** — the operational cost is immediate, the benefit hypothetical
- **Too few partitions**, capping parallelism permanently
- **Expecting global ordering** across partitions
- **Committing offsets before processing**, losing messages on a crash
- **Non-idempotent consumers**
- **Personal data in a retained log**, with no deletion strategy
- **No schema registry**, so a producer change breaks every consumer

---

## 19. Open Source Technologies

- **Apache Kafka**; **Redpanda** — a Kafka-compatible broker with simpler operations
- **Confluent Schema Registry**, **Apicurio** — schema enforcement
- **Kafka Connect**, **Debezium** — change data capture and integrations
- **Kafka Streams**, **Apache Flink** — stream processing
- **Managed**: Confluent Cloud, AWS MSK, Aiven

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Decide honestly whether your use case needs replay, or whether a queue would do.
- [ ] Work out your partition count from your required parallelism, not from a default.
- [ ] Trace what would happen to a consumer that processes a message and crashes before committing its offset.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Producers → topic (partitioned, replicated, retained) → many consumer groups
```

## 2. Request Flow

```text
Input       events appended to a partitioned topic, keyed for ordering
    ↓
Processing  retained on disk; each consumer group tracks its own offset
    ↓
Output      independent consumption, with replay available at any time
```

## 3. Real-World Usage

**Change data capture with Debezium** streams every database change into Kafka, so search indexes, caches and analytics stay current without polling. It is one of the clearest cases where the log model solves something a queue cannot.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A distributed, retained, replayable event log |
| **Why does it exist?** | Because queues delete events and only one consumer ever sees them |
| **Where does it belong?** | As the event backbone between many systems |
| **When should I use it?** | Replay, very high throughput, many consumers — not for background jobs |

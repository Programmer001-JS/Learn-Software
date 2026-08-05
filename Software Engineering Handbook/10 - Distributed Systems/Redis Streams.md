# Redis Streams

> **In one line —** Redis's answer to its own Pub/Sub problem: a persistent, replayable log with consumer groups, in a system you probably already run.

| | |
|---|---|
| **Category** | Message Log |
| **Architectural Layer** | Integration |
| **Introduced** | Redis 5.0 (2018) |
| **Related notes** | [Redis](../08%20-%20Databases%20and%20Data/Redis.md) · [Pub Sub](../08%20-%20Databases%20and%20Data/Pub%20Sub.md) · [Kafka](Kafka.md) · [Message Queues](Message%20Queues.md) |

---

## 1. Short Definition

*What is it?*

Redis Streams is an append-only log data type inside Redis. Entries are persisted, consumers track their position, and **consumer groups** distribute entries across workers with per-message acknowledgement.

---

## 2. Purpose

*What is its main purpose?*

To give Redis a messaging primitive that does not lose data — combining the durability and replay of a log with the operational simplicity of a system you already operate.

---

## 3. Problem

*What engineering problem does it solve?*

```text
REDIS PUB/SUB                         REDIS STREAMS
fire and forget                       persisted in Redis
subscriber offline → message LOST     offline consumer reads it later
no acknowledgement                    XACK per message
no replay                             replay from any ID
no work distribution                  consumer groups distribute entries
```

> [!IMPORTANT]
> If you have ever lost messages because a [Pub/Sub](../08%20-%20Databases%20and%20Data/Pub%20Sub.md) subscriber was restarting, Streams is the fix — and you do not need a new piece of infrastructure to adopt it.

---

## 4. Architecture Position

```text
Producer
    ↓  XADD stream * field value
┌──────── STREAM ────────┐
│ 1699-0 · 1700-0 · 1701-0 ... │  entries retained
└──────────┬──────────────┘
           ↓  XREADGROUP
┌─── consumer group "workers" ───┐
│  worker-1   worker-2   worker-3 │  each entry to ONE of them
└──────────┬─────────────────────┘
           ↓  XACK
     Pending entries list — anything unacknowledged
```

---

## 5. Consumer groups and the pending list

```text
XREADGROUP  delivers an entry to one consumer in the group
    ↓
The entry enters the PENDING ENTRIES LIST (PEL)
    ↓
XACK        → removed from pending, work confirmed
    ↓
Consumer crashes without acking
    ↓
The entry STAYS pending, visible via XPENDING
    ↓
XCLAIM / XAUTOCLAIM  → another consumer takes it over
```

> [!IMPORTANT]
> The pending entries list is what makes Streams reliable and is the piece people forget to implement. **Without a periodic `XAUTOCLAIM`, work from a crashed consumer sits pending forever** — delivered to nobody, silently.

---

## 6. Trimming — the operational essential

Streams live in RAM. They grow forever unless told otherwise.

```text
XADD stream MAXLEN ~ 10000 * field value
                    ↑ approximate trimming — much cheaper
```

> [!CAUTION]
> **An untrimmed stream will consume all available memory.** Unlike Kafka, which retains on cheap disk, Redis retains in expensive RAM. Set `MAXLEN` or `MINID` on every stream, from the first line of code.

---

## 7. Redis Streams vs the alternatives

| | Redis Streams | [Kafka](Kafka.md) | [RabbitMQ](RabbitMQ.md) | Pub/Sub |
|---|---|---|---|---|
| **Persistence** | Yes, in RAM | Yes, on disk | Yes, until ack | **No** |
| **Replay** | Yes, within retention | Yes | No | No |
| **Consumer groups** | Yes | Yes | Queues | No |
| **Per-message ack** | **Yes** (XACK) | Offsets | **Yes** | No |
| **Retention** | Limited by RAM | Days–weeks | Until consumed | None |
| **Operational cost** | **Very low** | High | Moderate | Very low |
| **Throughput** | Very high | Highest | High | Very high |

> [!TIP]
> **Redis Streams occupies a genuinely useful middle ground.** It gives you most of Kafka's model at a fraction of the operational cost — as long as your retention needs fit in RAM. For many systems that is hours or days of events, which is plenty.

---

## 8. Real World Example

- **Replacing Redis Pub/Sub** for WebSocket fan-out where missed messages mattered.
- **Background job queues** where an existing Redis is already deployed — no new broker needed.
- **Activity and audit feeds** with short retention.
- **[Celery](Celery.md) and [BullMQ](BullMQ.md)** use Redis for exactly this class of work, though with their own data structures.

---

## 9. Communication and Dependencies

- **A Redis instance**, ideally with AOF persistence enabled
- **A client supporting Streams** — `redis-py`, `ioredis`, Lettuce
- **Enough RAM** for the retention you configure
- **A claim loop** — periodic `XAUTOCLAIM` to recover abandoned entries

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Use Redis Streams when you already run Redis, need reliable delivery with acknowledgement, and your retention window fits comfortably in memory. It is frequently the right answer for a team that does not want to operate Kafka.

> [!CAUTION]
> - **Not for long retention** — RAM is far too expensive for weeks of events.
> - **Not for very large payloads** — keep entries small and store the body elsewhere.
> - **Not where durability must be absolute** — Redis persistence is good but weaker than a disk-based log; with AOF `everysec`, up to a second of writes can be lost on a hard failure.
> - **Not for rich routing** — there are no exchanges; that is RabbitMQ's job.

---

## 11. Advantages and Disadvantages

**Advantages**
- No new infrastructure if Redis is already deployed
- Persistent, with replay and acknowledgement
- Consumer groups distribute work correctly
- Extremely fast — sub-millisecond
- Simple mental model and a small API

**Disadvantages**
- Retention bounded by RAM cost
- Weaker durability guarantees than Kafka
- No routing beyond the stream key
- Requires disciplined trimming and claim handling
- Redis becomes an even more critical dependency

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Throughput** | Very high — hundreds of thousands of entries/second |
| **Latency** | Sub-millisecond |
| **Memory** | **The binding constraint** — entries × size × retention |
| **Persistence** | AOF `everysec` is the usual compromise between safety and speed |
| **Blocking reads** | `XREADGROUP BLOCK` avoids polling entirely |

---

## 13. Security Considerations

- **Redis must not be reachable from untrusted networks** — the historical default of no authentication caused mass compromises
- **Require authentication and use ACLs** to limit which streams a service may read or write
- **TLS** for connections crossing a network
- **Validate every entry** — a stream is untrusted input like any other
- **Retained entries are a data store**: a stream holding user events for 24 hours contains personal data with the same obligations as a database
- **Trimming is also a data-retention control**, not only a memory one

---

## 14. Mental Model

> [!NOTE]
> **Redis Streams is a whiteboard log with a sign-out sheet.**
>
> Entries are written in order and stay visible. Each worker takes an item and signs for it; if they walk away without signing, the item is still on the sheet and someone else can claim it. And because the whiteboard is finite, you wipe the oldest entries as you go.

---

## 15. Mini Architecture Diagram

```text
Producer ──XADD (MAXLEN ~10000)──► STREAM
                                      ↓ XREADGROUP
                          ┌── consumer group ──┐
                          │ w1    w2    w3     │
                          └─────────┬──────────┘
                                    ↓ XACK
                             Pending entries list
                                    ↓ XAUTOCLAIM
                            recovered by another worker
```

---

## 16. Complete Request Flow

```text
Order created
    ↓
XADD orders MAXLEN ~ 100000 * order_id 42 event created
    ↓
Entry gets an ID: 1699123456789-0
    ↓
XREADGROUP GROUP workers worker-1 BLOCK 5000 COUNT 10
    ↓
Entry delivered to worker-1 and added to the pending list
    ↓
Worker processes it — idempotently, because redelivery is possible
    ↓
XACK orders workers 1699123456789-0  → removed from pending
    ↓
─────────── worker crashes instead ───────────
Entry remains pending, with an idle time
    ↓
A periodic XAUTOCLAIM with min-idle-time 60000 reassigns it
    ↓
Another worker picks it up and completes it
    ↓
─────────── memory management ───────────
MAXLEN trims the oldest entries as new ones arrive
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> Redis Streams gives you persistence, replay and acknowledgement without a new broker — provided you trim aggressively and run a claim loop for abandoned work.

---

## 18. Common Mistakes

- **No `MAXLEN`** — the stream grows until Redis runs out of memory
- **No `XAUTOCLAIM` loop** — work from crashed consumers sits pending forever
- **Assuming `XREADGROUP` alone is reliable** — it is only reliable with `XACK` *and* claiming
- **Large payloads** in entries instead of references
- **Expecting Kafka-length retention** in RAM
- **Non-idempotent consumers**
- **Using it where Pub/Sub's fire-and-forget was actually fine**, adding complexity for nothing

---

## 19. Open Source Technologies

- **Redis** / **Valkey** — Streams is built in
- **redis-py**, **ioredis**, **Lettuce** — clients with group support
- **Redis Sentinel / Cluster** — availability for a now-critical dependency
- **BullMQ**, **Celery**, **RQ** — job frameworks built on Redis
- **Alternatives**: Kafka for long retention, RabbitMQ for routing, NATS JetStream

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Check whether every stream in your system has a `MAXLEN` and estimate its memory use.
- [ ] Run `XPENDING` on a stream and see whether anything has been stuck for a long time.
- [ ] Kill a consumer mid-processing and verify your claim loop recovers the entry.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Producer → Redis Stream (trimmed) → consumer group → XACK → pending list → XAUTOCLAIM
```

## 2. Request Flow

```text
Input       an entry appended to a stream
    ↓
Processing  delivered to one group member, tracked as pending until acknowledged
    ↓
Output      confirmed work, with abandoned entries reclaimable
```

## 3. Real-World Usage

Teams that outgrew **Redis Pub/Sub's message loss** but did not want to operate Kafka move to Streams. It is one of the clearest cases of a system growing a feature specifically to fix its own known limitation.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A persistent, replayable log with consumer groups inside Redis |
| **Why does it exist?** | Because Redis Pub/Sub loses messages and cannot distribute work |
| **Where does it belong?** | Between producers and workers, when Redis is already present |
| **When should I use it?** | Reliable messaging with short retention and minimal operational cost |

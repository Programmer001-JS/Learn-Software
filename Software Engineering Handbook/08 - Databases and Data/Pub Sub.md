# Pub/Sub

> **In one line —** a publisher broadcasts a message to a channel and never learns who received it; subscribers listen without the publisher knowing they exist.

| | |
|---|---|
| **Full name** | Publish/Subscribe |
| **Category** | Messaging Pattern |
| **Architectural Layer** | Integration |
| **Related notes** | [Redis](Redis.md) · [Message Queues](../10%20-%20Distributed%20Systems/Message%20Queues.md) · [Event Driven Architecture](../10%20-%20Distributed%20Systems/Event%20Driven%20Architecture.md) · [WebSockets](../04%20-%20Networking%20and%20Internet/WebSockets.md) · [Kafka](../10%20-%20Distributed%20Systems/Kafka.md) |

---

## 1. Short Definition

*What is it?*

Pub/Sub is a messaging pattern where senders (**publishers**) emit messages to a named **channel**, and any number of **subscribers** receive them. Neither side knows about the other.

---

## 2. Purpose

*What is its main purpose?*

To decouple who produces information from who reacts to it — so a new consumer can be added without touching the producer.

---

## 3. Problem

*What engineering problem does it solve?*

```text
DIRECT CALLS                          PUB/SUB
order_service:                        order_service:
    email.send(...)                       publish("order.created", data)
    analytics.track(...)
    inventory.reserve(...)            email · analytics · inventory · search
    search.index(...)                 all subscribe independently
    ↓                                     ↓
adding a consumer = editing           adding a consumer = new subscriber,
the producer                          producer untouched
one failure breaks the request        consumers fail independently
```

---

## 4. Architecture Position

```text
                 PUBLISHER
                     ↓
            ┌── CHANNEL: order.created ──┐
            ↓          ↓          ↓      ↓
        Email     Analytics   Search   Inventory
       consumer   consumer   consumer  consumer

        every subscriber gets EVERY message
```

> [!IMPORTANT]
> This is the defining difference from a [queue](../10%20-%20Distributed%20Systems/Message%20Queues.md): a **queue delivers each message to exactly one consumer** (work distribution); **pub/sub delivers every message to every subscriber** (broadcast).

---

## 5. Pub/Sub vs Queue vs Stream

| | Pub/Sub | Queue | Stream (Kafka) |
|---|---|---|---|
| **Delivery** | All subscribers | One consumer | All consumer groups |
| **Persistence** | Usually none | Until acknowledged | Retained for days |
| **Offline subscriber** | **Misses the message** | Gets it later | Reads from where it left off |
| **Replay** | No | No | Yes |
| **Use for** | Live notifications | Work distribution | Event log, analytics |

> [!CAUTION]
> **Classic Redis Pub/Sub is fire-and-forget.** A subscriber that is down, restarting or briefly disconnected misses the message permanently. There is no acknowledgement, no retry and no replay. This surprises people, and it is the single most important property to internalise.

---

## 6. Real World Example

- **[WebSocket](../04%20-%20Networking%20and%20Internet/WebSockets.md) fan-out** is the canonical use: user A's message must reach user B, who is connected to a *different* server instance. Redis Pub/Sub bridges them.
- **Cache invalidation across instances** — one server updates data and publishes "invalidate product:42" so every other instance drops its local copy.
- **Live dashboards** — metrics published continuously; if a viewer misses one tick, the next one corrects it.
- **Configuration reloads** — "settings changed, re-read them".

---

## 7. The pattern it makes possible

```text
User A ── Server 1 ──┐
                     ├── Redis Pub/Sub ──┐
User B ── Server 2 ──┘                   ↓
                              Server 2 pushes to User B's WebSocket
```

Without this, a chat application works perfectly on one server and breaks the moment you add a second — the two servers hold different connections and cannot reach each other's users.

---

## 8. Input, Processing, Output

**Input:** a channel name and a message payload.
**Processing:** the broker delivers a copy to every currently connected subscriber of that channel.
**Output:** each subscriber's callback fires — or nothing happens, if nobody is listening.

---

## 9. Communication and Dependencies

- **A broker** — Redis, NATS, MQTT, or a cloud service
- **A persistent connection** from each subscriber
- **Reconnection logic** — subscribers drop and must resubscribe
- **A fallback** for anything that must not be missed

---

## 10. Alternatives

```text
Redis Pub/Sub     simplest; no persistence, no delivery guarantee
    ↓
Redis Streams     persistent, consumer groups, replay — Redis's own answer
    ↓
NATS             very fast; JetStream adds persistence
    ↓
RabbitMQ         fanout exchange: pub/sub with acknowledgements and durability
    ↓
Kafka            retained, replayable event log; the heavyweight option
    ↓
Postgres LISTEN/NOTIFY   no extra system, but small payloads and no persistence
```

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use pub/sub for **broadcast of transient information**: live updates, cache invalidation, presence, configuration changes — anything where missing one message is recoverable.

> [!CAUTION]
> Do not use fire-and-forget pub/sub for anything that must not be lost: payments, order processing, emails, audit events. Those need a [queue](../10%20-%20Distributed%20Systems/Message%20Queues.md) with acknowledgements, or a persistent [stream](../10%20-%20Distributed%20Systems/Redis%20Streams.md).

---

## 12. Advantages and Disadvantages

**Advantages**
- Producers and consumers are fully decoupled
- New consumers added without touching producers
- Very low latency
- Simple mental model and simple API

**Disadvantages**
- **No delivery guarantee** in the classic form
- No persistence — offline subscribers miss everything
- No acknowledgement, no retry, no dead-letter handling
- Hard to debug — the producer does not know who consumed what
- A slow subscriber can be dropped by the broker

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Latency** | Sub-millisecond with Redis |
| **Throughput** | Very high; the broker only fans out |
| **Memory** | Minimal — messages are not stored |
| **Scaling** | Cost grows with subscribers × message rate |

---

## 14. Security Considerations

> [!CAUTION]
> **Channel names are not access control.** Any client with a connection to the broker can subscribe to any channel, including `user:42:private`. If subscribers are untrusted, the broker must never be reachable by them directly.

- **Never expose the broker to clients** — browsers connect to *your* server, which subscribes on their behalf and enforces authorisation
- **Publish identifiers, not sensitive payloads** — let each consumer fetch what it is authorised to see
- **Authenticate the broker** — Redis with no password on a reachable network is a known breach pattern
- **Validate messages on receipt** — a compromised publisher can emit anything

---

## 15. Mental Model

> [!NOTE]
> **Pub/Sub is a radio broadcast.**
>
> The station transmits without knowing who is listening. Anyone tuned in hears it; anyone who was out of the room missed it forever, and there is no repeat. That is fine for a traffic bulletin and completely wrong for a legal summons.

---

## 16. Mini Architecture Diagram

```text
Server 1 ──publish("room:42", msg)──► REDIS
                                        │
                     ┌──────────────────┼──────────────────┐
                     ↓                  ↓                  ↓
                 Server 1           Server 2           Server 3
                     ↓                  ↓                  ↓
              WebSocket users    WebSocket users    WebSocket users
```

---

## 17. Complete Request Flow

A chat message across a multi-server deployment:

```text
User A sends a message to Server 1
    ↓
Server 1 validates and authorises it
    ↓
Persisted to the database          ← durability lives HERE, not in pub/sub
    ↓
PUBLISH room:42 {message}
    ↓
Redis fans out to every subscribed server
    ↓
Server 2 receives it and pushes down User B's WebSocket
    ↓
Server 3 has no users in room 42 → ignores it
    ↓
Server 4 was restarting → MISSED IT
    ↓
Its users reconnect and load recent history from the database
```

> [!IMPORTANT]
> Note the last two steps. The design assumes messages can be missed, and recovers by reading the database on reconnect. **Pub/Sub delivers speed; the database delivers correctness.**

---

## 18. Key Takeaway

> [!IMPORTANT]
> Pub/Sub broadcasts to everyone currently listening and guarantees nothing to anyone who is not — use it for transient updates, never as the system of record.

---

## 19. Common Mistakes

- **Treating it as reliable delivery** — messages are lost, by design
- **Using it for payments, emails or orders** instead of a durable queue
- **No reconnection and resubscription logic**
- **Sensitive data in payloads** on channels that are not access-controlled
- **Exposing the broker to clients**
- **No fallback for missed messages** — reconnecting clients need a way to catch up
- **Confusing it with a queue** — broadcast is not work distribution

---

## 20. Open Source Technologies

- **Redis Pub/Sub**, **Redis Streams** — simple and persistent variants
- **NATS / JetStream** — very fast, with optional durability
- **RabbitMQ** fanout exchanges — pub/sub with acknowledgements
- **Apache Kafka** — retained, replayable event log
- **MQTT** (Mosquitto) — the IoT standard
- **PostgreSQL LISTEN/NOTIFY** — when you already have Postgres and needs are modest

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] List everything in your system published over pub/sub and mark which items would cause a real problem if lost.
- [ ] Test what happens when a subscriber restarts mid-traffic.
- [ ] Design a catch-up path for one channel so reconnecting clients recover missed messages.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Publisher → channel → all connected subscribers
                          ↓
              (durability lives in the database, not here)
```

## 2. Request Flow

```text
Input       a message published to a named channel
    ↓
Processing  broadcast to every currently connected subscriber
    ↓
Output      callbacks fire; anyone disconnected receives nothing, ever
```

## 3. Real-World Usage

**Every multi-server WebSocket application** needs this. Chat, collaborative editing and live dashboards all break when a second instance is added, and Redis Pub/Sub is the standard bridge between instances.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Broadcast messaging where publishers and subscribers do not know each other |
| **Why does it exist?** | To decouple producers of information from reactors to it |
| **Where does it belong?** | Between services and between server instances |
| **When should I use it?** | Transient broadcasts — never for messages that must not be lost |

# RabbitMQ

> **In one line —** the classic task queue: per-message acknowledgement, flexible routing, and a message disappears once it is done.

| | |
|---|---|
| **Category** | Message Broker |
| **Architectural Layer** | Integration |
| **Protocol** | AMQP 0-9-1 |
| **Written in** | Erlang |
| **Related notes** | [Message Queues](Message%20Queues.md) · [Kafka](Kafka.md) · [Background Workers](Background%20Workers.md) · [Celery](Celery.md) |

---

## 1. Short Definition

*What is it?*

RabbitMQ is a message broker built around **queues and acknowledgements**. A producer publishes to an exchange, the exchange routes to queues, and a consumer acknowledges each message — at which point it is removed.

---

## 2. Purpose

*What is its main purpose?*

To distribute individual units of work reliably, with fine-grained control over routing, retries and per-message acknowledgement.

---

## 3. Problem

*What engineering problem does it solve?*

Work must be handed to exactly one worker, must not be lost if that worker dies, and must sometimes be routed to different consumers based on its content.

---

## 4. Architecture Position

```text
Producer
    ↓  publish with a routing key
EXCHANGE          ← decides where the message goes
    ↓  bindings
Queue A · Queue B · Queue C
    ↓  one consumer per message
Workers
    ↓  ack / nack
Message removed, or requeued, or dead-lettered
```

---

## 5. Exchanges — the routing model

This is RabbitMQ's distinguishing feature.

```text
DIRECT    routing key matches exactly
          "email.welcome" → the email queue

TOPIC     wildcard patterns
          "order.*.created"  ·  "order.eu.#"
          → the most useful in practice

FANOUT    every bound queue receives a copy
          → pub/sub behaviour

HEADERS   routes on message headers rather than a routing key
          → rarely needed
```

> [!TIP]
> A **topic exchange** covers almost every real routing requirement. Producers publish `order.eu.created`; one consumer binds `order.#` for auditing, another binds `order.eu.*` for regional processing — and neither producer nor consumer knows about the other.

---

## 6. Acknowledgements

```text
Consumer receives a message
    ↓
Processes it
    ↓
ack   → removed from the queue
nack  → requeued, or sent to the dead-letter exchange
crash without acking → REDELIVERED to another consumer
```

> [!CAUTION]
> **Never use automatic acknowledgement** (`auto_ack=True`). It acknowledges on delivery rather than on completion, so a worker crash loses the message silently. Manual acknowledgement after successful processing is the whole point of choosing RabbitMQ.

---

## 7. Prefetch — the setting that matters operationally

```text
prefetch = unlimited (default in some clients)
    ↓
One worker grabs 10,000 messages
    ↓
Other workers sit idle; that worker crashes and everything is redelivered

prefetch = 1..10
    ↓
Work distributes evenly; a crash affects few messages
```

> [!TIP]
> Set `prefetch` to a small number for slow tasks and a larger one for fast ones. Leaving it unbounded is the most common RabbitMQ misconfiguration.

---

## 8. Dead-letter exchanges

```text
Message rejected N times, or expires
    ↓
Dead-letter exchange
    ↓
Dead-letter queue → inspect, fix, replay
```

Combined with a TTL, this also implements **delayed retry**: publish to a queue with a TTL and no consumer, and let it dead-letter back into the working queue after the delay.

---

## 9. Real World Example

- **[Celery](Celery.md)** uses RabbitMQ as its canonical broker.
- **Order processing, notifications, image pipelines** — the standard workloads.
- **Microservice choreography** where different services need different subsets of events, via topic routing.

---

## 10. RabbitMQ vs Kafka

> [!IMPORTANT]
> These are frequently compared and are built for different jobs.

| | RabbitMQ | [Kafka](Kafka.md) |
|---|---|---|
| **Model** | Queue — message consumed and deleted | Log — messages retained |
| **Replay** | No | **Yes** |
| **Routing** | **Rich** (exchanges, topics, patterns) | Partitions and topics only |
| **Per-message ack** | **Yes** | Offset commits |
| **Throughput** | Tens of thousands/sec | **Millions/sec** |
| **Best for** | **Task distribution** | Event streaming, analytics |

```text
"Send this email"                        → RabbitMQ
"Every click event, retained for a week" → Kafka
```

---

## 11. Communication and Dependencies

- **A client library** — `pika`, `amqplib`, Spring AMQP
- **Erlang runtime** — RabbitMQ's own dependency
- **Clustering and quorum queues** for high availability
- **The management plugin** — an unusually good web UI

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use RabbitMQ for background job processing with meaningful routing needs, per-message acknowledgement, and moderate throughput. For most applications this is the right shape.

> [!CAUTION]
> - **Not for event streaming or analytics** — messages are deleted after consumption, so there is nothing to replay.
> - **Not for millions of messages per second** — that is Kafka's territory.
> - **Not if you want zero operations** — SQS or a managed offering removes the Erlang cluster from your responsibilities.

---

## 13. Advantages and Disadvantages

**Advantages**
- Very flexible routing
- Reliable per-message acknowledgement and redelivery
- Dead-letter exchanges and delayed retry built in
- Excellent management UI and observability
- Mature and stable, with clients everywhere

**Disadvantages**
- No replay — a consumed message is gone
- Lower throughput ceiling than a log-based broker
- Erlang clustering is unfamiliar operational territory
- Queues that grow very large degrade performance
- More concepts to learn than a simple queue service

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Throughput** | Tens of thousands of messages/second per node |
| **Latency** | Sub-millisecond in-cluster |
| **Memory** | Large backlogs are held in memory until paged to disk — watch this |
| **Durable queues** | Persistence costs throughput; choose deliberately |

> [!CAUTION]
> **A queue that grows into the millions is a warning sign, not a buffer.** RabbitMQ is designed for queues that stay short. Sustained growth means consumers cannot keep up, and performance degrades as the backlog is paged to disk.

---

## 15. Security Considerations

- **Change the default `guest/guest` credentials** — they are localhost-only by default, and that default is frequently overridden in containers
- **Never expose the management UI or AMQP port** to the internet
- **Use vhosts** to isolate applications and environments
- **Per-user permissions** on exchanges and queues
- **TLS** between clients and broker
- **Validate every message payload** — messages are untrusted input
- **The dead-letter queue holds real payloads** and is usually the least secured part of the setup

---

## 16. Mental Model

> [!NOTE]
> **RabbitMQ is a post room with sorting rules.**
>
> Post arrives with an address; the sorting rules decide which pigeonhole it goes in. Each item is collected by exactly one person, who signs for it. If they drop it without signing, it goes back in the pigeonhole for someone else. Once signed for, it is gone — there is no archive.

---

## 17. Mini Architecture Diagram

```text
Producer
    ↓ routing key: order.eu.created
┌──── TOPIC EXCHANGE ────┐
│  order.#      → audit queue
│  order.eu.*   → eu processing queue
│  order.*.created → notification queue
└─────────┬──────────────┘
          ↓
     Consumers (ack per message)
          ↓ repeated failure
   Dead-letter exchange → DLQ
```

---

## 18. Complete Request Flow

```text
API publishes "order.eu.created" with a durable, persistent message
    ↓
Topic exchange evaluates bindings
    ↓
Copies routed to three queues
    ↓
Each queue delivers to ONE of its consumers, respecting prefetch
    ↓
Consumer processes the message — idempotently
    ↓
ack → removed from that queue
    ↓
Consumer crashes before acking → connection drops → REDELIVERED elsewhere
    ↓
Fails repeatedly → nack without requeue → dead-letter exchange → DLQ → alert
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> RabbitMQ excels at routing individual tasks with per-message acknowledgement — and once a message is acknowledged it is gone, so it is a work queue rather than an event log.

---

## 20. Common Mistakes

- **`auto_ack=True`** — silent message loss on a worker crash
- **Unbounded prefetch** — uneven distribution and large redelivery batches
- **No dead-letter exchange** — poison messages retried forever
- **Non-durable queues or non-persistent messages** where durability was assumed
- **Treating it as an event log** and expecting replay
- **Letting queues grow into the millions**
- **Default credentials exposed** in a container deployment

---

## 21. Open Source Technologies

- **RabbitMQ** and its management plugin
- **Celery**, **Spring AMQP**, **MassTransit** — framework integrations
- **Quorum queues** — the modern replicated queue type
- **Shovel** and **Federation** plugins — cross-datacentre links
- **Alternatives**: NATS (simpler, faster), Kafka (log), SQS (managed)

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Check the prefetch setting on your consumers and observe distribution with several workers running.
- [ ] Verify that acknowledgement happens after processing, not on delivery.
- [ ] Kill a worker mid-task and confirm the message is redelivered rather than lost.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Producer → exchange → queues → consumers (ack) → DLQ on failure
```

## 2. Request Flow

```text
Input       a message with a routing key
    ↓
Processing  routed by the exchange, delivered to one consumer, acknowledged
    ↓
Output      work completed and the message removed, or dead-lettered
```

## 3. Real-World Usage

**Celery with RabbitMQ** is the standard Python background-processing stack. The combination of routing flexibility and per-message acknowledgement is exactly what task distribution needs — and exactly what Kafka does not provide.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A message broker built on exchanges, queues and acknowledgements |
| **Why does it exist?** | To distribute individual tasks reliably with flexible routing |
| **Where does it belong?** | Between producers and background workers |
| **When should I use it?** | Task queues with routing needs — not event streaming |

# AWS SQS

> **In one line —** a fully managed queue with effectively unlimited scale and no servers to operate; the boring choice, which is usually the right one on AWS.

| | |
|---|---|
| **Full name** | Amazon Simple Queue Service |
| **Category** | Managed Message Queue |
| **Architectural Layer** | Integration |
| **Related notes** | [Message Queues](Message%20Queues.md) · [RabbitMQ](RabbitMQ.md) · [Lambda](../12%20-%20Cloud%20Architecture/Lambda.md) · [Serverless](Serverless.md) · [IAM](../12%20-%20Cloud%20Architecture/IAM.md) |

---

## 1. Short Definition

*What is it?*

SQS is AWS's managed queue service. You create a queue through an API call, send and receive messages over HTTPS, and AWS operates everything underneath.

---

## 2. Purpose

*What is its main purpose?*

To provide reliable, scalable queuing with **zero operational burden** — no cluster, no brokers, no capacity planning, no patching.

---

## 3. Problem

*What engineering problem does it solve?*

```text
SELF-HOSTED BROKER                    SQS
provision servers                     create a queue (one API call)
configure clustering                  send and receive
monitor and patch                     AWS handles everything else
plan capacity                             ↓
handle failover                       scales to any volume automatically
    ↓
real, ongoing operational cost
```

---

## 4. Architecture Position

```text
Producer  (API, Lambda, any service with IAM permission)
    ↓  SendMessage over HTTPS
┌──────── SQS QUEUE ────────┐
│  redundant across AZs      │
│  visibility timeout        │
│  retention up to 14 days   │
└────────────┬───────────────┘
             ↓  ReceiveMessage (long polling)
     Consumers · Lambda · ECS tasks
             ↓  DeleteMessage
     Dead-letter queue after N failures
```

---

## 5. The visibility timeout — the concept to understand

SQS does not delete a message when it is received. It **hides** it.

```text
ReceiveMessage
    ↓
Message becomes INVISIBLE for the visibility timeout (default 30 s)
    ↓
Consumer processes it
    ↓
DeleteMessage  → gone
    ↓
─────────── or ───────────
Consumer crashes, or takes longer than the timeout
    ↓
Message becomes VISIBLE again → delivered to another consumer
```

> [!CAUTION]
> **If processing takes longer than the visibility timeout, the message is redelivered while you are still working on it** — and two consumers now handle the same job. Set the timeout above your realistic worst-case processing time, or extend it during processing with `ChangeMessageVisibility`.

---

## 6. Standard vs FIFO queues

| | **Standard** | **FIFO** |
|---|---|---|
| **Throughput** | Effectively unlimited | 300–3,000 msg/s |
| **Ordering** | Best effort — **not guaranteed** | Guaranteed within a group |
| **Duplicates** | **Possible** — at-least-once | Exactly-once processing within a 5-minute window |
| **Cost** | Lower | Higher |
| **Use for** | Most work | Order-sensitive operations |

> [!IMPORTANT]
> **Standard queues can and do deliver duplicates and reorder messages.** This is not an edge case — it is the documented behaviour, and it is the price of the scale. Your consumers must be idempotent. FIFO queues reduce the problem but cap throughput sharply.

---

## 7. Long polling

```text
SHORT POLLING   returns immediately, often empty
                → constant requests, higher cost, wasted calls

LONG POLLING    WaitTimeSeconds = 20
                → the call waits until a message arrives or 20 s elapse
                → fewer requests, lower cost, lower latency
```

> [!TIP]
> **Always enable long polling.** Short polling is the default in some SDK paths and produces both higher bills and worse latency — an unusual combination of being worse on every axis.

---

## 8. Dead-letter queues

```text
maxReceiveCount = 5
    ↓
A message received 5 times without deletion
    ↓
Automatically moved to the dead-letter queue
    ↓
Inspect, fix, redrive
```

SQS has a built-in **redrive** operation to move messages back once the bug is fixed — a genuinely useful feature that self-hosted setups usually have to build.

---

## 9. Real World Example

- **Lambda triggered by SQS** — the canonical serverless worker pattern; AWS scales the consumers automatically.
- **Decoupling microservices** on AWS without running a broker.
- **Buffering traffic spikes** — the queue absorbs a burst that would overwhelm downstream services.
- **SQS behind SNS** — SNS fans out to several SQS queues, giving pub/sub plus durable per-consumer queues.

---

## 10. Communication and Dependencies

- **IAM permissions** — access control is IAM policy, not a username and password
- **The AWS SDK** or any HTTPS client
- **CloudWatch** for queue depth and message age metrics
- **A dead-letter queue**, configured explicitly

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use SQS for essentially any queuing need on AWS. The absence of operational work is worth a great deal, and the scale ceiling is far above what most systems require.

> [!CAUTION]
> - **No rich routing** — one queue, one purpose. Use SNS or EventBridge in front if you need fan-out.
> - **No replay** — a deleted message is gone; use Kinesis or Kafka for an event log.
> - **Vendor lock-in** — the API is AWS-specific, though the abstraction is thin enough to wrap.
> - **Latency** — HTTPS-based, so slightly higher than an in-cluster broker. Rarely relevant for background work.
> - **256 KB message limit** — larger payloads go to S3 with a pointer in the message.

---

## 12. Advantages and Disadvantages

**Advantages**
- No servers, no clustering, no patching
- Effectively unlimited scale on standard queues
- Redundant across availability zones by default
- Dead-letter queues and redrive built in
- Pay per request; nothing to run when idle
- IAM integration for fine-grained access control

**Disadvantages**
- No routing, no replay
- Duplicates and reordering on standard queues
- FIFO throughput is limited
- 256 KB message size limit
- AWS-specific API
- Per-request cost adds up at very high volume

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Latency** | Tens of milliseconds — HTTPS round trip |
| **Throughput** | Unlimited (standard); 3,000/s (FIFO with batching) |
| **Cost** | Per million requests — batch to reduce it |
| **Retention** | Up to 14 days |

> [!TIP]
> Use **batch operations** (`SendMessageBatch`, up to 10 messages) and long polling together. Both reduce request count, which is what you are actually billed for.

---

## 14. Security Considerations

- **IAM policies are the access control** — grant `sqs:SendMessage` and `sqs:ReceiveMessage` separately, to the specific queue ARN
- **Never use a wildcard queue policy** — a publicly writable queue is a resource-exhaustion and injection vector
- **Server-side encryption (SSE-SQS or KMS)** for message bodies at rest
- **Validate every message** — a message from another service is untrusted input
- **Avoid sensitive data in messages**; store it elsewhere and send a reference
- **The dead-letter queue holds real payloads** and needs the same protection as the main queue
- **VPC endpoints** keep traffic off the public internet

---

## 15. Mental Model

> [!NOTE]
> **SQS is a shared mailbox that the postal service maintains for you.**
>
> You do not own the building, wire the alarms or hire the staff. You drop letters in and collect them. Each letter is taken by one person, and if they walk off without it the letter reappears in the box a short time later — which is why you must not act twice on the same letter.

---

## 16. Mini Architecture Diagram

```text
Producers
    ↓ SendMessageBatch
SQS queue  (multi-AZ, retention up to 14 days)
    ↓ ReceiveMessage — long polling, visibility timeout
Lambda / ECS consumers
    ↓ DeleteMessage
Dead-letter queue after maxReceiveCount → redrive when fixed
```

---

## 17. Complete Request Flow

```text
API sends a message (batched with up to 9 others)
    ↓
Stored redundantly across availability zones
    ↓
Consumer long-polls with WaitTimeSeconds=20
    ↓
Message received → becomes invisible for the visibility timeout
    ↓
Consumer checks its idempotency key — already processed? → delete and stop
    ↓
Work performed
    ↓
DeleteMessage → permanently removed
    ↓
─────────── crash instead ───────────
Visibility timeout expires → message reappears → another consumer takes it
    ↓
After 5 attempts → dead-letter queue → CloudWatch alarm
    ↓
Bug fixed → redrive → messages returned to the main queue
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> SQS removes all operational work from queuing, and pays for it with duplicates and possible reordering — set the visibility timeout above your worst-case processing time, and make consumers idempotent.

---

## 19. Common Mistakes

- **Visibility timeout shorter than processing time** — duplicate concurrent processing
- **Short polling** left enabled, raising cost and latency
- **No dead-letter queue**, so poison messages cycle forever
- **Assuming ordering** on a standard queue
- **Non-idempotent consumers**
- **Forgetting `DeleteMessage`** — the message simply reappears
- **Overly broad IAM or queue policies**
- **Payloads over 256 KB** without the S3 pointer pattern

---

## 20. Open Source Technologies

- **ElasticMQ**, **LocalStack** — SQS-compatible local development
- **Celery**, **BullMQ** — support SQS as a broker
- **AWS SDK** batch and long-polling helpers
- **Alternatives**: Google Pub/Sub, Azure Service Bus, RabbitMQ, NATS

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Compare your visibility timeout with the actual p99 processing time of your slowest consumer.
- [ ] Confirm long polling is enabled and check the effect on your request count.
- [ ] Verify every queue has a dead-letter queue with an alarm on its depth.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Producer → SQS (multi-AZ) → consumers → DLQ → redrive
```

## 2. Request Flow

```text
Input       a message sent over HTTPS with IAM authorisation
    ↓
Processing  hidden by a visibility timeout, deleted on success, redelivered on failure
    ↓
Output      work completed idempotently, or parked in a dead-letter queue
```

## 3. Real-World Usage

**SQS triggering Lambda** is the default serverless background-processing pattern on AWS: no broker, no worker fleet, and consumer scaling handled by the platform. It is the clearest example of managed infrastructure removing a whole category of work.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A fully managed, highly scalable message queue |
| **Why does it exist?** | So teams do not operate broker clusters |
| **Where does it belong?** | Between producers and workers on AWS |
| **When should I use it?** | Nearly all AWS queuing — with idempotent consumers, always |

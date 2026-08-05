# Event Driven Architecture

> **In one line —** services announce what happened instead of telling each other what to do, which decouples them completely and makes the system much harder to reason about.

| | |
|---|---|
| **Category** | Architecture Pattern |
| **Architectural Layer** | System |
| **Related notes** | [Message Queues](Message%20Queues.md) · [Kafka](Kafka.md) · [Pub Sub](../08%20-%20Databases%20and%20Data/Pub%20Sub.md) · [Microservices](Microservices.md) · [Domain Driven Design](../07%20-%20Backend%20Design%20Patterns/Domain%20Driven%20Design.md) |

---

## 1. Short Definition

*What is it?*

An architecture where components communicate by publishing **events** — statements of fact about something that already happened — rather than by calling each other directly.

```text
COMMAND   "send a welcome email"      → an instruction to a known recipient
EVENT     "a user registered"         → a fact; the publisher does not care who listens
```

---

## 2. Purpose

*What is its main purpose?*

To let new behaviour be added without modifying the code that triggers it. The producer of an event never changes when a new consumer appears.

---

## 3. Problem

*What engineering problem does it solve?*

```text
DIRECT CALLS                          EVENT-DRIVEN
order_service:                        order_service:
    email.send(...)                       publish(OrderPlaced)
    inventory.reserve(...)
    analytics.track(...)              email · inventory · analytics · loyalty
    loyalty.award(...)                each subscribes independently
    ↓                                     ↓
adding "loyalty points" means         adding it means writing one new consumer
editing the order service             the order service is untouched
one slow dependency slows the order   consumers fail without failing the order
```

---

## 4. Architecture Position

```text
Producer service
    ↓  publishes an EVENT (past tense, a fact)
Event broker  (Kafka, RabbitMQ, EventBridge)
    ↓
Consumer A   Consumer B   Consumer C
    ↓            ↓            ↓
their own    their own    their own
databases    databases    databases
```

---

## 5. Events describe the past

```text
✓ OrderPlaced · PaymentReceived · UserRegistered · InvoiceIssued
✗ SendEmail · CreateInvoice · ChargeCard
```

> [!IMPORTANT]
> **If the name is an instruction, it is a command, not an event** — and a command sent through an event bus is a remote procedure call with extra latency and no error handling. Events are past tense because they cannot be refused: the thing already happened.

---

## 6. Choreography vs orchestration

```text
CHOREOGRAPHY                          ORCHESTRATION
each service reacts to events         a coordinator drives the sequence
    ↓                                     ↓
no central control                    explicit, visible flow
maximum decoupling                    easier to reason about and to debug
NOBODY KNOWS THE WHOLE FLOW           the coordinator is a dependency
```

> [!TIP]
> **Choreography for simple fan-out; orchestration for multi-step business processes.** A five-step order flow implemented purely as choreography is genuinely difficult to understand — the sequence exists only as an emergent property of who happens to subscribe to what.

---

## 7. The saga pattern

Distributed transactions are not available across services, so long-running processes are modelled as a sequence of local transactions with **compensating actions**.

```text
Reserve stock  ─── OK ──►  Charge card ─── FAILS
      ↑                          │
      └──── compensate: release stock ◄───┘
```

> [!CAUTION]
> **Compensation is not rollback.** Money already charged is refunded, not un-charged; an email already sent cannot be unsent. Every step of a saga needs a designed compensating action, and some steps have none — which is a design constraint, not an implementation detail.

---

## 8. Eventual consistency

```text
Order placed
    ↓  immediately
Order exists in the order service
    ↓  200 ms later
Inventory reflects the reservation
    ↓  2 seconds later
Analytics counts it
    ↓
For that window, different services disagree about reality
```

> [!IMPORTANT]
> **The user will see this.** They place an order and their dashboard does not show it yet. The system is not broken; it is eventually consistent — but the interface must be designed for that, with optimistic updates or explicit pending states.

---

## 9. Real World Example

- **E-commerce order flows** — the archetypal use: one `OrderPlaced` event drives inventory, payment, shipping, email and analytics.
- **[Change data capture](Kafka.md)** — database changes become events that keep search indexes and caches current.
- **Audit and compliance** — an event log is a natural record of everything that happened.
- **[Domain events](../07%20-%20Backend%20Design%20Patterns/Domain%20Driven%20Design.md)** inside a single application, without any distribution at all.

---

## 10. The outbox pattern

```text
✗ NAIVE
   BEGIN; save order; COMMIT;
   publish(OrderPlaced)          ← crash here: the order exists, the event never happened

✓ OUTBOX
   BEGIN; save order; INSERT INTO outbox (event); COMMIT;
       ↓
   A relay process reads the outbox and publishes
       ↓
   At-least-once publication, atomic with the business write
```

> [!TIP]
> This is the single most important implementation detail in event-driven systems, and the one most often skipped. Without it, events are silently lost whenever a process dies at the wrong moment.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use events when several independent things must happen in response to one occurrence, when consumers are added over time, and when temporal decoupling is genuinely valuable.

> [!CAUTION]
> Do not adopt event-driven architecture as a default. It converts a readable synchronous call into a distributed flow with no stack trace, and debugging becomes "which of eleven consumers did not react, and why?"
>
> Within a single application, **domain events give most of the decoupling benefit with none of the distribution cost.**

---

## 12. Advantages and Disadvantages

**Advantages**
- Producers and consumers evolve independently
- New behaviour added without touching existing services
- Failures isolated — a broken consumer does not fail the original action
- Natural audit trail
- Absorbs load spikes through buffering

**Disadvantages**
- **No single place shows the whole flow**
- Debugging requires distributed tracing, not a stack trace
- Eventual consistency leaks into the user experience
- Event schema changes are breaking changes for every consumer
- Duplicate delivery requires idempotency everywhere
- Ordering is not guaranteed across partitions or queues

---

## 13. Event versioning

```text
Producers and consumers deploy independently
    ↓
An event's shape WILL need to change
    ↓
✓ Only add optional fields
✓ Never remove or rename a field
✓ Never change a field's meaning
✓ Publish v2 alongside v1 during a transition
```

> [!CAUTION]
> An event schema is a **public API with unknown consumers**. Once published, you cannot know who depends on which field. A schema registry turns "we broke someone" into a build-time failure.

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Producer latency** | Improves — no waiting for downstream services |
| **End-to-end latency** | Worse — hops through a broker |
| **Throughput** | Excellent; consumers scale independently |
| **Debugging cost** | **Significantly higher** — this is the real price |

---

## 15. Security Considerations

> [!CAUTION]
> **An event bus is a broadcast channel, and any consumer with access sees everything on it.** Publishing a `UserRegistered` event containing an email address, phone number and address means every subscribing service now holds personal data — including ones with no need for it.

- **Publish identifiers; let consumers fetch what they are authorised to see**
- **Authorise per topic** — a service subscribes only to what it needs
- **Authenticate producers** — anyone who can publish can fabricate facts, and consumers act on them
- **Validate every event** against a schema before acting on it
- **A retained event log is a data store** with full GDPR obligations, and deleting one person's events from an immutable log is genuinely hard
- **Trace context in every event** — without it, incident investigation is guesswork

---

## 16. Mental Model

> [!NOTE]
> **Direct calls are phoning each department in turn. Events are a company-wide announcement.**
>
> The announcement is faster, and anyone who cares can act on it — including departments you have never met. But nobody has the complete picture of what happens after the announcement, and if one department was on holiday, nothing tells you their part was skipped.

---

## 17. Mini Architecture Diagram

```text
Order service
   │ BEGIN: save order + write to outbox; COMMIT
   ↓
 Relay ──► Event broker (OrderPlaced)
                │
    ┌───────────┼───────────┬───────────┐
    ↓           ↓           ↓           ↓
 Inventory   Payment    Email      Analytics
    ↓           ↓
 own DB     own DB      ← each service owns its own data
    ↓
 saga compensations on failure
```

---

## 18. Complete Request Flow

```text
POST /orders
    ↓
BEGIN: order saved + OrderPlaced written to the outbox; COMMIT
    ↓
201 returned in ~30 ms — no downstream service was called
    ↓
Relay publishes OrderPlaced to the broker
    ↓
┌─ Inventory reserves stock → publishes StockReserved
├─ Payment charges the card → FAILS → publishes PaymentFailed
├─ Email queues a confirmation
└─ Analytics records the event
    ↓
Inventory consumes PaymentFailed → compensates → releases stock
    ↓
Order service consumes PaymentFailed → marks the order failed
    ↓
The user's dashboard updates — seconds after the original response
    ↓
Every consumer is idempotent, because redelivery is normal
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> Events decouple producers from consumers completely — at the cost of no single place showing the whole flow, and eventual consistency that users can see.

---

## 20. Common Mistakes

- **Commands disguised as events** (`SendEmail` published to a bus)
- **No outbox pattern** — events lost when a process dies after commit
- **Sensitive data in event payloads**
- **Breaking schema changes** with unknown consumers
- **Non-idempotent consumers**
- **Choreography for complex multi-step flows** nobody can trace
- **No distributed tracing** — incidents become unsolvable
- **Adopting it by default** where a function call would do

---

## 21. Open Source Technologies

- **Kafka**, **RabbitMQ**, **NATS**, **Redis Streams** — brokers
- **Debezium** — change data capture, an alternative to a hand-written outbox
- **Schema Registry**, **Apicurio** — enforce event contracts
- **Temporal**, **Camunda** — saga orchestration with visible state
- **OpenTelemetry**, **Jaeger** — distributed tracing, non-optional here
- **AsyncAPI** — documenting event-driven interfaces

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Take one event in your system and list every consumer. Could you name them all without looking?
- [ ] Check whether any event is published outside the transaction that produced it.
- [ ] Design the compensating action for one step of a multi-service flow — and find one step that has none.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Producer → outbox → broker → many independent consumers → their own databases
```

## 2. Request Flow

```text
Input       something happened in one service
    ↓
Processing  recorded atomically, published as a fact, consumed independently
    ↓
Output      several systems updated eventually, with compensations on failure
```

## 3. Real-World Usage

**E-commerce order processing** is the standard example: one `OrderPlaced` event drives payment, inventory, shipping, email and analytics. It is also the standard cautionary tale — teams that chose choreography for the whole flow often cannot answer "why did this order not ship?" without reading five services' logs.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Communication through published facts rather than direct calls |
| **Why does it exist?** | So new behaviour can be added without changing existing services |
| **Where does it belong?** | Between services that must evolve independently |
| **When should I use it?** | Multiple independent reactions to one occurrence — not as a default |

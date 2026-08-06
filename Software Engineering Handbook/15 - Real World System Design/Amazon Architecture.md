# Amazon Architecture

> **In one line —** the company that turned an organisational rule about how teams communicate into a technical architecture, and then discovered the internal platform was more valuable than the shop.

| | |
|---|---|
| **Category** | Case Study *(system design)* |
| **Architectural Layer** | Whole-system |
| **Related notes** | [Microservices](../10%20-%20Distributed%20Systems/Microservices.md) · [Event Driven Architecture](../10%20-%20Distributed%20Systems/Event%20Driven%20Architecture.md) · [Transactions](../08%20-%20Databases%20and%20Data/Transactions.md) · [Design Patterns](../07%20-%20Backend%20Design%20Patterns/Design%20Patterns.md) · [Cache](../08%20-%20Databases%20and%20Data/Cache.md) · [AWS Architecture](../12%20-%20Cloud%20Architecture/AWS%20Architecture.md) |

---

## 1. The System in One Page

*What are we designing?*

```text
A shop with hundreds of millions of products and hundreds of millions of
customers, that must never lose an order and never oversell an item.

    BROWSE      search, categories, recommendations — READ-DOMINATED
    PRODUCT     one page assembled from dozens of independent services
    CART        survives sessions, devices and days
    CHECKOUT    inventory, payment, fraud, tax, shipping — all must agree
    FULFIL      warehouses, packing, carriers, tracking
    RETURN      the reverse of all of it, which is genuinely harder
```

> [!IMPORTANT]
> **Amazon's architecture is the clearest example in the industry of an organisational decision producing a technical outcome.** The early-2000s mandate — every team exposes its data *only* through a service interface, with no shared databases and no back doors — was a management instruction. Service-oriented architecture, and eventually AWS, were its consequences. If you take one thing from this note, take the direction of causation: **the org chart shapes the architecture, whether or not anyone intends it.**

---

## 2. Requirements and constraints

```text
SCALE (orders of magnitude)
    hundreds of millions of active customers
    hundreds of millions of product listings
    tens of millions of orders on a peak day
    a product page assembled from DOZENS of service calls
    thousands of independent services

CONSTRAINTS
    latency directly affects revenue — measurably
    an order, once placed, must never be lost
    inventory must not be oversold, across many channels at once
    peak days are ~10× normal, and known in advance
    physical fulfilment means software errors have real-world cost
```

---

## 3. The mandate, and what it forced

```text
"All teams will expose their data and functionality through service interfaces.
 Teams must communicate with each other through these interfaces.
 There will be no other form of interprocess communication allowed:
 no direct linking, no direct reads of another team's data store,
 no shared-memory model, no back doors whatsoever."
```

```text
WHAT THIS FORCED, whether teams wanted it or not
    every interface became a CONTRACT, versioned and documented
    every team became responsible for its own uptime and on-call
    every service had to be independently deployable
    every interface was, by construction, ready to be sold externally
    ↓
    which is how AWS happened
```

> [!TIP]
> **"No shared database" is the single most consequential rule in that list.** Two services reading the same tables are one service with two deployment pipelines: neither can change its schema, neither can be reasoned about alone, and a slow query in one degrades the other. Most failed microservice migrations are failed *database* separations. If you adopt nothing else from Amazon, adopt this constraint — and notice it is enforceable in a monolith too, as module boundaries.

---

## 4. Two-pizza teams and service ownership

```text
A TEAM
    small enough to be fed by two pizzas
    owns its services END TO END: design, build, deploy, on-call, cost
    "you build it, you run it"
    ↓
CONSEQUENCE
    the person woken at 3 a.m. is the person who wrote the code
    → which changes what people build, permanently
```

> [!IMPORTANT]
> **"You build it, you run it" is a reliability mechanism disguised as an ownership policy.** When a separate operations team carries the pager, the incentive to add a timeout, a fallback or a useful log message is weak. When the author carries it, those things appear without a mandate. This is a cheaper way to improve reliability than most engineering initiatives, and it requires no new technology.

---

## 5. The product page: composition under a latency budget

```text
ONE product page needs
    core product data · price · availability · reviews · ratings
    recommendations · images · Q&A · seller info · shipping estimate
    personalisation · experiments
    ↓
DOZENS of service calls, in parallel, under a total budget of ~100-200 ms
```

```text
THE DISCIPLINE THAT MAKES IT POSSIBLE
    every call has a TIMEOUT measured in tens of milliseconds
    every call has a defined FALLBACK
    → recommendations slow?     show a generic list
    → reviews slow?             show the rating only
    → personalisation slow?     show the default
    NOTHING blocks the page. The page always renders.
```

> [!IMPORTANT]
> **With thirty dependencies, something is always slow — so "always render something" must be a design rule rather than an aspiration.** If each of thirty services has a 99.9% success rate and any failure breaks the page, the page succeeds 97% of the time. With fallbacks, it succeeds essentially always and is occasionally less good. This arithmetic is why fallback design, not service reliability, is the load-bearing practice in composed systems.

---

## 6. Reads and writes are radically different here

```text
READ PATH (99% of traffic)                WRITE PATH (the 1% that matters)
browse, search, product pages              add to cart, place order, pay
    ↓                                          ↓
cache everything, aggressively            strict correctness required
eventual consistency is fine              idempotent, durable, auditable
CDN + edge + multi-layer caches            queued, retried, reconciled
optimise for THROUGHPUT                   optimise for NEVER LOSING IT
```

> [!TIP]
> **Almost every commerce system has this asymmetry, and treating both paths identically is the common mistake.** The read path wants caching, replicas and denormalisation; the write path wants transactions, idempotency and durability. Designing one architecture that compromises between them gives you a read path that is slower than it needs to be and a write path that is less safe than it should be.

---

## 7. Inventory: the hardest consistency problem in the shop

```text
"Is this item in stock?"  is deceptively difficult
    ↓
    stock is spread across many warehouses
    sold simultaneously on several channels
    physically miscounted sometimes
    reserved by carts that may be abandoned
```

```text
THE PRACTICAL APPROACH
    DISPLAY availability approximately, from a cache
        → "In stock" may be a few seconds stale, and that is acceptable
    RESERVE atomically at checkout
        → a conditional decrement; the authoritative decision point
    OVERSELL occasionally, and handle it as a BUSINESS process
        → apologise, refund, reorder — this is cheaper than global locking
```

> [!CAUTION]
> **Strong global consistency on inventory display would require coordinating every warehouse on every page view, which is not affordable.** So the design accepts rare overselling and compensates commercially. This is worth internalising: **some correctness problems are cheaper to resolve with a business process than with a distributed transaction.** The engineering skill is knowing which — money must never be wrong; a stock count can be, if there is a defined way to make the customer whole.

---

## 8. Checkout: a saga, not a transaction

```text
Placing an order touches: inventory, payment, fraud, tax, shipping, notification
    ↓
These are separate services with separate databases.
There is NO distributed transaction across them.
```

```text
SAGA PATTERN — a sequence of local transactions with COMPENSATIONS
    reserve inventory        → compensate: release it
    authorise payment        → compensate: void the authorisation
    fraud check              → compensate: cancel
    create shipment          → compensate: cancel shipment
    ↓
Any step fails → run the compensations for the completed steps, backwards
```

> [!IMPORTANT]
> **The order must be *accepted* durably before all of this completes — which is why an order confirmation is not the same as a completed order.** The write is captured immediately (durable, idempotent, queued), then the saga runs asynchronously. This is why you get "we're processing your order" and occasionally, later, "unfortunately we could not fulfil it". That user experience is a direct consequence of the architecture, and it is the right trade: never lose the order, resolve the details afterwards.

---

## 9. Dynamo, and what it cost

```text
THE PROBLEM (early 2000s)
    the shopping cart must ALWAYS accept a write
    a relational primary that is unavailable = lost revenue, immediately
    ↓
DYNAMO'S CHOICE: availability over consistency
    always accept the write, resolve conflicts later
    ↓
CONSEQUENCE, made concrete
    a cart edited on two devices during a partition MERGES
    → a deleted item could reappear
    → judged better than refusing the write
```

> [!TIP]
> **The Dynamo paper is worth reading for the reasoning, not the implementation.** It made the availability-versus-consistency trade explicit and product-driven: for a cart, accepting a write always is worth an occasional resurrected item; for a payment, it is not. DynamoDB is its managed descendant, and the general lesson survives — **pick the consistency model per dataset, based on what the business actually loses in each failure mode.** See [Transactions](../08%20-%20Databases%20and%20Data/Transactions.md).

---

## 10. Architecture diagram

```text
                            customer
                                │
                    ┌───────────▼────────────┐
                    │  CDN / edge             │  images, static, some pages
                    └───────────┬────────────┘
                                │
                    ┌───────────▼────────────┐
                    │  API / page composition │  ← the latency budget lives here
                    │  parallel fan-out       │
                    │  timeout + fallback     │
                    │  on EVERY call          │
                    └──┬──┬──┬──┬──┬──┬──────┘
       ┌───────────────┘  │  │  │  │  └──────────────┐
       ▼                  ▼  ▼  ▼  ▼                 ▼
   ┌────────┐  ┌────────┐ ┌──────┐ ┌────────┐  ┌──────────┐
   │ catalog│  │ pricing│ │review│ │ recs   │  │ inventory│   ← each team owns
   │ own DB │  │ own DB │ │own DB│ │ own DB │  │  own DB  │     its own store
   └────────┘  └────────┘ └──────┘ └────────┘  └──────────┘     NO SHARING
                                                     │
   ═══════════════ WRITE PATH ═══════════════════════▼══════════
                                          ┌──────────────────┐
   cart ──► ORDER SERVICE ──durable──────►│  event bus /     │
            (idempotent, accepted first)  │  queue           │
                                          └────┬──┬──┬──┬────┘
                                               ▼  ▼  ▼  ▼
                                    payment fraud tax shipping
                                          (SAGA, with compensations)
                                               │
                                               ▼
                                    warehouse / fulfilment systems
                                               │
                                               ▼
                                     notification · tracking
```

---

## 11. Complete request flow

```text
─────────────── browsing ───────────────
Customer opens a product page
    ↓
Edge serves images and static assets; the page shell is requested
    ↓
Composition layer fans out to ~30 services IN PARALLEL, each with a
tens-of-milliseconds timeout
    ↓
Recommendations exceed their 40 ms budget → FALLBACK to a generic popular list
    ↓
The reviews service returns the aggregate rating but not the review text in time
→ FALLBACK: rating shown, reviews load asynchronously
    ↓
Page renders in ~150 ms. Two services were degraded. The customer noticed nothing.
    ↓
Availability shown from a cache — possibly seconds stale, deliberately
    ↓
─────────────── add to cart ───────────────
Cart write goes to a store chosen for AVAILABILITY over consistency
    ↓
It always succeeds, even during a partition — losing a cart write loses revenue
    ↓
─────────────── checkout ───────────────
Order submitted with an IDEMPOTENCY KEY generated by the client
    ↓
The customer double-clicks; the retry carries the same key → ONE order
    ↓
Order DURABLY ACCEPTED and acknowledged: "we're processing your order"
    ↓
The saga begins, asynchronously:
    1. reserve inventory        → atomic conditional decrement → OK
    2. authorise payment        → OK
    3. fraud check              → OK
    4. tax and shipping         → OK
    5. create fulfilment order  → OK
    ↓
Confirmation email sent asynchronously
    ↓
─────────────── a step fails ───────────────
Payment authorisation is declined at step 2
    ↓
COMPENSATION: release the inventory reservation
    ↓
Customer notified, asked for another payment method
    ↓
No distributed transaction was ever held open
    ↓
─────────────── the rare oversell ───────────────
Two channels sell the last unit within the same second
    ↓
The atomic decrement lets one through; a warehouse miscount lets the other
    ↓
Handled as a BUSINESS process: apologise, refund, offer an alternative
    ↓
Cheaper than the global coordination that would have prevented it
    ↓
─────────────── peak day ───────────────
Traffic is 10× normal, and the date was known months ahead
    ↓
Capacity pre-scaled; caches pre-warmed; non-essential features disabled
    ↓
Load shedding protects checkout at the expense of, say, recommendations
    ↓
PRIORITY IS EXPLICIT: the write path is protected, the read path degrades
```

---

## 12. Key design decisions, and what they cost

| Decision | Bought | Cost |
|---|---|---|
| **No shared databases** | Genuine team independence | Every join becomes an API call |
| **Two-pizza teams owning on-call** | Reliability incentives aligned | Duplicated effort across teams |
| **Timeout + fallback on every call** | The page always renders | Every dependency needs a designed failure mode |
| **Cached, approximate availability** | Fast pages at enormous read volume | Occasional overselling |
| **Sagas instead of distributed transactions** | Independent services, high availability | Compensation logic, and visible intermediate states |
| **Availability over consistency for carts** | Never lose a cart write | Merge anomalies customers can see |
| **Idempotency keys throughout** | Safe retries on every write | Extra state to keep |
| **Service interfaces as products** | AWS | Interface discipline no monolith requires |

---

## 13. What transfers to a normal system

```text
DIRECTLY APPLICABLE, at any size
    ✓ NO SHARED DATABASES between components — enforceable in a monolith too
    ✓ timeout + fallback on every external call
    ✓ idempotency keys on every write that matters
    ✓ separate the read path (cache hard) from the write path (be strict)
    ✓ sagas with compensation, instead of distributed transactions
    ✓ accept an order durably first, resolve the details asynchronously
    ✓ let the people who write the code carry the pager
    ✓ decide per dataset whether availability or consistency wins

DO NOT COPY WITHOUT AMAZON'S SCALE AND STAFFING
    ✗ thousands of services
    ✗ a service per team when you have three teams
    ✗ Dynamo-style stores unless you have that specific availability need
    ✗ splitting a monolith before the team boundaries hurt
```

> [!TIP]
> **The most valuable idea here is available to a team of five: enforce module boundaries with no shared tables, and put a timeout and fallback on every outbound call.** That is most of the architectural benefit with none of the distributed-systems cost. Splitting into services is what you do when *organisational* coordination becomes the bottleneck — not when the code gets large.

---

## 14. Security Considerations

> [!CAUTION]
> **A commerce system's threat model is dominated by fraud and abuse rather than intrusion.** Stolen cards, account takeover, refund and return fraud, fake reviews, price-scraping and coupon abuse are continuous, adversarial and economically motivated. This shapes the architecture: fraud checks are a service in the checkout saga, not an afterthought.

- **PCI scope minimised** — card data isolated in a tokenising payment service, ideally never touching your systems
- **Account takeover is the main customer-facing risk**; see [Login System Architecture](Login%20System%20Architecture.md)
- **Fraud scoring as a first-class step** in the order saga, with compensation if it declines
- **Idempotency keys are also an abuse control** — they prevent replay of a payment authorisation
- **Prices, discounts and taxes computed server-side, always** — never trust a client-supplied amount
- **Rate limiting on search and product endpoints**, which are scraped continuously
- **Seller and third-party integrations are a supply chain risk** in a marketplace
- **Audit trails on every order state change**, because disputes and regulators require them

---

## 15. Mental Model

> [!NOTE]
> **Amazon is a department store where every counter is a separate business.**
>
> The shoe counter will not let the electronics counter into its stockroom — if electronics need to know about shoes, they ask at the counter, and they take an answer or they take a shrug. The shop floor is arranged so a customer sees a full display even when three counters are unstaffed. And critically: the order is written into the book the moment you hand it over, before anyone checks the stockroom — because losing the order is unforgivable, while telling you an hour later that the item is unavailable is merely disappointing.

---

## 16. Key Takeaway

> [!IMPORTANT]
> Amazon's transferable lessons are organisational as much as technical: no shared databases, teams that run what they build, a timeout and fallback on every call, idempotent writes accepted durably before the details are resolved, and consistency chosen per dataset by what the business loses.

---

## 17. Common Mistakes When Copying This

- **Splitting into services while keeping a shared database** — the worst of both models
- **Microservices adopted for code size** rather than for team coordination cost
- **No fallbacks**, so thirty dependencies produce a fragile page
- **Distributed transactions attempted** across services, instead of sagas
- **No idempotency keys**, so a double-click creates two orders
- **Strong consistency demanded on inventory display**, making pages slow for a rare benefit
- **Treating read and write paths identically**
- **Trusting client-supplied prices or totals**
- **Separate operations team on the pager**, removing the incentive to build reliable code
- **Sagas with no compensation logic**, leaving orders stuck in intermediate states
- **Copying the org structure without the autonomy** — two-pizza teams that cannot deploy independently are just small teams with meetings

---

## 18. Open Source Technologies

- **The Dynamo paper (2007)** — read it for the reasoning; **DynamoDB** and **Cassandra** are its descendants
- **Apache Kafka**, **RabbitMQ**, **NATS** — the event backbone for order flows
- **Temporal**, **Cadence**, **Camunda** — implement sagas and compensations without hand-rolling them
- **resilience4j**, **Polly**, **tenacity** — timeouts, circuit breakers, fallbacks in application code
- **Envoy**, **Istio** — enforce timeouts and retries at the infrastructure layer
- **Redis**, **Varnish** — the read-path caching that makes the page fast
- **OpenTelemetry**, **Jaeger** — without tracing, a thirty-service page is undebuggable
- **Elasticsearch / OpenSearch** — product search
- **Debezium** — change data capture, the honest way to share data without sharing a database

---

## 19. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 20. Workbook Exercise

- [ ] Find any two components in your system that read the same tables. That is one component, not two.
- [ ] Count the outbound calls on your most important page. How many have a timeout and a fallback?
- [ ] Check whether double-submitting your most important form creates two records.
- [ ] Pick a multi-step business process and write down its compensation for each step.
- [ ] For each dataset, decide whether you would rather accept a write or refuse it during a partition.
- [ ] Ask who is on call for the code you wrote last month.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
edge cache → parallel page composition (timeout + fallback per call) → services with OWN databases
write path: idempotent durable accept → event bus → saga with compensations → fulfilment
```

## 2. Request Flow

```text
Input       a read-dominated browse workload, plus a small number of critical writes
    ↓
Processing  parallel composition with fallbacks; writes accepted durably then resolved by saga
    ↓
Output      a page that always renders, and an order that is never lost
```

## 3. Real-World Usage

Amazon's influence on the industry runs through two channels: the service-boundary mandate that became the standard argument for microservices, and the Dynamo paper that made the availability-versus-consistency trade a design decision rather than a database property. Both are frequently mis-applied — the first by teams splitting code without splitting databases, the second by teams choosing eventual consistency for data that needed to be exact.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A composed, service-per-team commerce system with a cached read path and a saga-driven write path |
| **Why is it built this way?** | Because an organisational mandate forbade shared data, and revenue depends on both speed and never losing an order |
| **Where does it belong?** | As a reference for service boundaries, fallbacks and eventual consistency by choice |
| **What should I take from it?** | No shared databases, fallbacks everywhere, idempotent writes, sagas, and per-dataset consistency |

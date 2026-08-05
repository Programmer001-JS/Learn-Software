# Microservices

> **In one line —** splitting one application into many independently deployable services, which solves an organisational problem and creates a great many technical ones.

| | |
|---|---|
| **Category** | Architecture Pattern |
| **Architectural Layer** | System |
| **Related notes** | [Monolith](Monolith.md) · [Modular Monolith](Modular%20Monolith.md) · [Event Driven Architecture](Event%20Driven%20Architecture.md) · [Domain Driven Design](../07%20-%20Backend%20Design%20Patterns/Domain%20Driven%20Design.md) · [Kubernetes](../13%20-%20DevOps%20and%20Delivery/Kubernetes.md) |

---

## 1. Short Definition

*What is it?*

An architecture where one application is split into small services, each owning its own data, deployed independently, and communicating over the network.

---

## 2. Purpose

*What is its main purpose?*

> [!IMPORTANT]
> **The real purpose is organisational, not technical.** Microservices let many teams deploy independently without coordinating releases. Almost every technical claim made for them — scalability, resilience, technology choice — is achievable in a well-built [monolith](Monolith.md). Independent deployment by independent teams is the one that is not.

If you have one team, you are paying the entire cost and collecting almost none of the benefit.

---

## 3. Problem

*What engineering problem does it solve?*

```text
LARGE ORGANISATION, ONE CODEBASE
50 developers, one deployment pipeline
    ↓
Every release coordinates 50 people's changes
One team's bug blocks everyone's deploy
Merge conflicts, release trains, coupled schedules
    ↓
Deployment frequency drops as the team grows
```

---

## 4. Architecture Position

```text
Client
    ↓
API Gateway            auth, routing, rate limiting
    ↓
┌──────────┬──────────┬──────────┐
│ Orders   │ Payments │ Users    │  ← each owns its data
│ own DB   │ own DB   │ own DB   │
└─────┬────┴─────┬────┴─────┬────┘
      └──── message broker ──┘
              ↓
      observability: tracing, logs, metrics
```

> [!IMPORTANT]
> **A shared database is not microservices.** If two services read and write the same tables, they are coupled at the deepest possible level: neither can change its schema independently, which was the entire point. This is the most common way the pattern is adopted in name only.

---

## 5. What you must have first

```text
Automated deployment per service        or releases become worse, not better
Centralised logging                     or debugging means SSH into 12 machines
Distributed tracing                     or you cannot follow one request
Monitoring and alerting per service     or failures are invisible
Container orchestration                 or you are hand-managing dozens of processes
Service discovery                       or hard-coded addresses
API versioning discipline               or one change breaks three services
```

> [!CAUTION]
> **These are prerequisites, not follow-up work.** Adopting microservices without them produces a distributed monolith: all of the operational complexity, none of the independence.

---

## 6. What actually gets harder

| In a monolith | In microservices |
|---|---|
| A function call | A network call that can fail, retry or time out |
| A database transaction | A [saga](Event%20Driven%20Architecture.md) with compensating actions |
| A stack trace | Distributed tracing across services |
| A join | An API call, or duplicated data |
| Refactoring across modules | A coordinated multi-service release |
| Running locally | Docker Compose with twelve containers |

> [!CAUTION]
> **Losing transactions is the underestimated one.** An operation that was `BEGIN ... COMMIT` becomes a multi-step saga with compensating actions, eventual consistency, and a failure mode for every step. That is not a small tax; it is a permanent change to how you write business logic.

---

## 7. Service boundaries

```text
✗ BY TECHNICAL LAYER      "database service", "API service"
                          → every feature touches all of them

✗ BY ENTITY               a service per database table
                          → chatty, and nothing is independent

✓ BY BUSINESS CAPABILITY  Ordering · Payments · Shipping · Identity
                          → a feature usually lives in one service
```

> [!TIP]
> [Bounded contexts](../07%20-%20Backend%20Design%20Patterns/Domain%20Driven%20Design.md) are the best available guide to where the lines should go. A boundary is correct when most changes touch one service — if a typical feature requires coordinated releases across four services, the boundaries are wrong.

---

## 8. Real World Example

- **Netflix and Amazon** are the canonical adopters — both with thousands of engineers and a genuine coordination problem.
- **Amazon's rule** that every team communicates only through service interfaces predates the term, and produced AWS.
- **The counter-examples matter more**: several well-known companies have publicly consolidated microservices back into monoliths after finding the operational cost outweighed the benefit at their scale.

---

## 9. When To Use / When NOT To Use

> [!TIP]
> Use microservices when you have **multiple teams that need to deploy independently**, clear domain boundaries, and the operational maturity listed in section 5. Those three conditions together are the honest test.

> [!CAUTION]
> Do not adopt them for:
> - **A small team** — you get the cost and none of the benefit
> - **Scalability** — a monolith scales horizontally too, and usually further than people assume
> - **A new product** — the boundaries are unknown, and moving them later is far harder than moving a module
> - **Because large companies do** — Netflix's constraints are not yours
>
> **Start with a [modular monolith](Modular%20Monolith.md).** It gives you clean boundaries with the option to split later, and splitting a well-modularised monolith is a manageable project. Splitting a tangled one is not, and neither is un-splitting premature services.

---

## 10. Advantages and Disadvantages

**Advantages**
- Independent deployment per team
- Independent scaling of hot components
- Fault isolation — one service failing need not take everything down
- Technology choice per service
- Smaller, more comprehensible codebases

**Disadvantages**
- **Enormous operational complexity**
- No distributed transactions
- Network calls fail, retry and time out
- Debugging requires distributed tracing
- Data duplication and eventual consistency
- Local development becomes a project in itself
- Higher infrastructure cost
- Cross-service refactoring is very expensive

---

## 11. Performance Impact

| Aspect | Impact |
|---|---|
| **Latency** | Worse — every internal call is a network hop |
| **Chatty designs** | 10 internal calls per request is a real and common failure |
| **Scaling** | Better — scale only what is hot |
| **Resource use** | Higher — each service carries its own runtime |
| **Debugging time** | Substantially higher |

> [!IMPORTANT]
> **A network call is roughly a million times slower than a function call.** A request that fans out to five services and back has spent more time on the network than a monolith would have spent doing the whole job.

---

## 12. Security Considerations

> [!CAUTION]
> **The internal network is not a security boundary.** The classic microservice mistake is assuming that "it is only reachable internally" means it is safe — which fails the moment any single service is compromised, and gives an attacker free movement across everything.

- **mTLS between services** — zero trust, not perimeter trust
- **Authorise in every service**, not only at the gateway
- **Each service gets its own credentials** with least privilege
- **The attack surface multiplies** — twelve services means twelve dependency trees, twelve configurations, twelve sets of endpoints
- **Propagate identity, do not re-derive it** — pass a verified token and validate it at each hop
- **Secrets management per service**, not one shared credential

---

## 13. Mental Model

> [!NOTE]
> **A monolith is a house; microservices are a street of houses.**
>
> Each household is independent: they renovate whenever they like without asking anyone. But now everything shared — water, post, disputes — requires infrastructure and coordination that a single house simply did not need. That is worth it for a hundred families. It is absurd for one.

---

## 14. Mini Architecture Diagram

```text
                    Client
                       ↓
                 API Gateway  (auth, rate limit, routing)
                       ↓
   ┌───────────┬───────────┬───────────┐
 Orders     Payments    Shipping    Identity
   │            │           │           │
 own DB      own DB      own DB      own DB
   └────────────┴─── broker ─────┴───────┘
                       ↓
      distributed tracing · centralised logs · per-service metrics
```

---

## 15. Complete Request Flow

```text
POST /orders arrives at the gateway
    ↓
Gateway authenticates and forwards with a trace ID
    ↓
Order service validates and saves to ITS OWN database
    ↓
Publishes OrderPlaced via the outbox pattern
    ↓
201 returned — no downstream service was awaited
    ↓
Payment service consumes it → charges → publishes PaymentSucceeded
    ↓
Inventory consumes it → reserves stock
    ↓
Shipping consumes PaymentSucceeded → creates a shipment
    ↓
─────────── failure ───────────
Payment fails → publishes PaymentFailed
    ↓
Inventory compensates: releases the stock
    ↓
Order service marks the order failed
    ↓
Investigating any of this requires the trace ID across five services
```

---

## 16. Key Takeaway

> [!IMPORTANT]
> Microservices solve an organisational problem — independent deployment by independent teams — and you pay for it in distributed transactions, network failures and debugging. Start with a modular monolith.

---

## 17. Common Mistakes

- **Adopting them with one team**
- **A shared database** between services — coupling at the deepest level
- **Boundaries drawn by technical layer** rather than business capability
- **No distributed tracing** — incidents become unsolvable
- **Synchronous chains** of five services, where one slow hop fails the request
- **A distributed monolith** — services that must be released together
- **Trusting the internal network**
- **Splitting before the domain is understood**

---

## 18. Open Source Technologies

- **Kubernetes**, **Docker** — orchestration and packaging
- **Istio**, **Linkerd** — service mesh with automatic mTLS
- **Kong**, **Envoy**, **Traefik** — API gateways
- **OpenTelemetry**, **Jaeger**, **Tempo** — distributed tracing
- **Kafka**, **RabbitMQ** — asynchronous communication
- **Temporal** — saga orchestration with visible state

---

## 19. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 20. Workbook Exercise

- [ ] Count the teams that need to deploy independently in your organisation. Is it more than one?
- [ ] Take one recent feature and count how many services it would have required a change to.
- [ ] Trace one request end to end. Could you do it without a trace ID?

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Client → gateway → independent services (each with its own database) → broker
```

## 2. Request Flow

```text
Input       a request routed to one service
    ↓
Processing  handled locally; other services react asynchronously via events
    ↓
Output      a fast response, with the rest of the work eventually consistent
```

## 3. Real-World Usage

**Amazon's rule that teams communicate only through service interfaces** is the origin story of this pattern — and of AWS. It was an organisational constraint that produced a technical architecture, which is the correct order and the one most adopters reverse.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An application split into independently deployable services owning their own data |
| **Why does it exist?** | So many teams can deploy without coordinating |
| **Where does it belong?** | Large organisations with real coordination costs |
| **When should I use it?** | Multiple teams, clear boundaries, operational maturity — otherwise a modular monolith |

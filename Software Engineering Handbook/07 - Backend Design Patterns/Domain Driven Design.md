# Domain Driven Design

> **In one line —** model the software on the business, using the same words the business uses, so the code says what the domain experts say.

| | |
|---|---|
| **Category** | Design Approach |
| **Architectural Layer** | Domain / whole system |
| **Abbreviation** | DDD *(not [Documentation Driven Development](../01%20-%20Foundation/07%20-%20Documentation%20Driven%20Development.md))* |
| **Origin** | Eric Evans, *Domain-Driven Design* (2003) |
| **Related notes** | [Clean Architecture](Clean%20Architecture.md) · [Services](Services.md) · [Microservices](../10%20-%20Distributed%20Systems/Microservices.md) · [Event Driven Architecture](../10%20-%20Distributed%20Systems/Event%20Driven%20Architecture.md) |

---

## 1. Short Definition

*What is it?*

DDD is an approach where the structure and language of the code deliberately mirror the business domain. Its central claim: for complex domains, the hard part is understanding the business, not the technology.

---

## 2. Purpose

*What is its main purpose?*

To eliminate the translation layer between what domain experts say and what the code contains — because every translation is a place where meaning is lost.

---

## 3. Problem

*What engineering problem does it solve?*

```text
Domain expert says:  "when a policy lapses, outstanding claims are frozen"
Code says:           status_id = 3; UPDATE claims SET flag = true

    ↓
Nobody can verify the code against the rules.
Rules live in one person's head. That person leaves.
```

---

## 4. Ubiquitous Language

The foundational practice, and the cheapest one to adopt.

> [!IMPORTANT]
> **One vocabulary, used by everyone, everywhere.** If the business says "policy lapses", the class is `Policy` with a method `lapse()` — not `updateStatus(3)`. The same words appear in conversation, documentation, code, database tables and API endpoints.

```text
BAD                             GOOD
UserManager.processData()       Policy.lapse()
status_id = 3                   PolicyStatus.LAPSED
flag_b = true                   Claim.freeze()
```

The moment a developer says "what the business calls a lapse, we call status 3", knowledge has started leaking.

---

## 5. Strategic DDD — the part that matters most

### Bounded Context

The same word means different things in different parts of a business.

```text
                    "CUSTOMER"

SALES CONTEXT           BILLING CONTEXT        SUPPORT CONTEXT
lead source             payment method         ticket history
deal stage              tax ID                 satisfaction score
assigned rep            credit limit           preferred channel
```

> [!IMPORTANT]
> These are **not** one `Customer` table with forty nullable columns. They are three models in three contexts, connected by identifiers. Forcing them into one shared model is the single most common cause of a data model nobody can change.

**Bounded contexts are the strongest guide to [microservice](../10%20-%20Distributed%20Systems/Microservices.md) boundaries there is** — better than splitting by technical layer or by team org chart.

### Context Map

How contexts relate: shared kernel, customer/supplier, anti-corruption layer. An **anti-corruption layer** translates between your model and an external or legacy one, so their concepts do not leak into yours.

---

## 6. Tactical DDD — the building blocks

| Concept | Meaning |
|---|---|
| **Entity** | Has identity that persists through change — `Order #42` |
| **Value Object** | Defined by its values, immutable — `Money(10, EUR)`, `Address` |
| **Aggregate** | A cluster of objects treated as one unit, with one root |
| **Aggregate Root** | The only entry point; enforces the cluster's invariants |
| **Repository** | One per aggregate, not per table — see [Repository Pattern](Repository%20Pattern.md) |
| **Domain Event** | Something that happened — `OrderPlaced`, `PolicyLapsed` |
| **Domain Service** | A rule that belongs to no single entity |

### The aggregate rule

```text
Order (aggregate root)
  └── OrderLine
  └── OrderLine

✓ order.add_line(...)          → the root enforces "max 50 lines", "total ≤ limit"
✗ order.lines.append(...)      → the invariant is bypassed
✗ orderLineRepo.save(line)     → no repository for a non-root entity
```

> [!TIP]
> **One transaction, one aggregate.** If an operation must modify two aggregates atomically, either your boundaries are wrong, or you need eventual consistency through a domain event.

---

## 7. Value objects — the underused idea

```python
# Primitive obsession
def transfer(amount: float, currency: str, from_: str, to: str): ...
transfer(100, "EUR", account_b, account_a)   # ← arguments swapped, compiles fine

# Value objects
def transfer(amount: Money, from_: AccountId, to: AccountId): ...
transfer(Money(100, EUR), account_b, account_a)   # ← type error
```

Value objects are cheap to adopt, need no framework, and remove a whole class of bugs. If you take one thing from DDD into a codebase, take this.

---

## 8. Real World Example

- **Insurance, banking, logistics, healthcare** — domains where rules are genuinely complex and long-lived.
- **Microservice boundaries** derived from bounded contexts rather than from technical layers.
- **Event sourcing and CQRS** frequently accompany DDD, though neither is required by it.

---

## 9. When To Use / When NOT To Use

> [!TIP]
> Use full DDD when the domain is genuinely complex, the rules are the product, domain experts are available to talk to, and the system will live for years.

> [!CAUTION]
> **DDD is heavily over-applied.** For CRUD applications, content sites and simple SaaS products, aggregates and value objects add ceremony without protecting anything — there are no invariants to protect.
>
> The honest split: **adopt Ubiquitous Language and Value Objects everywhere** — they are nearly free. Adopt aggregates, bounded contexts and the full tactical toolkit only where complexity justifies them.

---

## 10. Advantages and Disadvantages

**Advantages**
- Code readable by domain experts
- Business rules concentrated and enforceable
- Bounded contexts give principled service boundaries
- Invariants enforced by aggregates rather than by convention
- Knowledge survives staff turnover

**Disadvantages**
- Steep learning curve, and easy to do superficially
- Substantial boilerplate for simple domains
- Requires access to domain experts — without them it is guesswork
- Aggregate boundaries are hard to get right and expensive to change
- Frequently reduced to "folders named domain/" with none of the substance

---

## 11. Performance Impact

| Aspect | Impact |
|---|---|
| **Runtime** | Negligible |
| **Aggregate loading** | Loading a whole aggregate for one field can be wasteful |
| **One transaction per aggregate** | Pushes cross-aggregate consistency to events |
| **CQRS** | Often introduced precisely to separate read performance from write modelling |

---

## 12. Security Considerations

> [!IMPORTANT]
> **Aggregates are an excellent place for invariants that are also security properties.** "An order's total may not exceed the customer's credit limit" enforced inside the aggregate root cannot be bypassed by any caller — no controller, job or script can construct an invalid state.

- **Bounded contexts limit blast radius** — a compromised support service should not be able to reach billing data
- **Anti-corruption layers** stop untrusted external models from entering your domain unchecked
- **Domain events must not carry sensitive payloads** — publish identifiers, let consumers fetch what they are authorised to see

---

## 13. Mental Model

> [!NOTE]
> **DDD is drawing the map with the locals rather than from a satellite photo.**
>
> A satellite shows every road accurately and tells you nothing about which ones matter. The locals tell you that these two districts are entirely separate worlds even though they touch — and that distinction, not the geometry, is what you actually need to navigate.

---

## 14. Mini Architecture Diagram

```text
        ┌─────────── Sales Context ───────────┐
        │  Lead · Opportunity · Customer      │
        └──────────────┬──────────────────────┘
                       │  customer_id
        ┌──────────────┴─── Billing Context ──┐
        │  Invoice · Payment · Customer       │
        │      Aggregate: Invoice             │
        │        └── InvoiceLine              │
        └─────────────────────────────────────┘
                       │  domain events
        ┌──────────────┴─── Support Context ──┐
        │  Ticket · Customer                  │
        └─────────────────────────────────────┘

        same word, three models, connected by identity
```

---

## 15. Complete Request Flow

```text
"Place an order for 3 items"
    ↓
Application service loads the Order AGGREGATE through its repository
    ↓
order.add_line(sku, 3)
    ↓
The aggregate root enforces its invariants:
    max lines? credit limit? product available in this region?
    ↓
Invariant violated → domain exception in domain language
    ↓
One transaction, one aggregate → persisted
    ↓
Domain event OrderPlaced published after commit
    ↓
Billing context reacts, in its own transaction, eventually consistent
```

---

## 16. Key Takeaway

> [!IMPORTANT]
> DDD says the hard part is the business, not the technology — adopt its language and value objects everywhere, and its heavy machinery only where the domain is genuinely complex.

---

## 17. Common Mistakes

- **Applying tactical DDD to a CRUD app** — aggregates protecting nothing
- **"DDD" as folder names**, with no ubiquitous language and no invariants
- **One shared `Customer` model** across every context
- **Aggregates too large**, loading enormous object graphs
- **Modifying several aggregates in one transaction**
- **Anemic entities** — data classes with all rules in services, which is not DDD
- **Doing it without domain experts**, which is inventing a domain rather than modelling one

---

## 18. Open Source Technologies

- **Axon**, **EventStoreDB** — event sourcing infrastructure
- **MediatR** (.NET), **Python `dataclass(frozen=True)`** — value objects and use-case handlers
- **EventStorming** — the workshop technique for discovering bounded contexts
- Eric Evans, *Domain-Driven Design*; Vaughn Vernon, *Implementing Domain-Driven Design*

---

## 19. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 20. Workbook Exercise

- [ ] Write down five terms your business uses and check whether they appear literally in your code.
- [ ] Find one entity that means different things in different parts of your system, and describe the two contexts.
- [ ] Replace one primitive parameter pair (amount + currency) with a value object.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Bounded Context
    ↓
Aggregate (root enforces invariants)
    ↓
Entities · Value Objects
    ↓
Repository (one per aggregate)
    ↓
Domain events → other contexts
```

## 2. Request Flow

```text
Input       an operation expressed in domain language
    ↓
Processing  aggregate loaded, invariants enforced, one transaction
    ↓
Output      a persisted state change plus domain events
```

## 3. Real-World Usage

**Bounded contexts are how well-designed microservice boundaries are actually chosen.** Teams that split by technical layer end up with services that cannot change independently; teams that split by context get services that own their own language and data.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Designing software around the business domain and its language |
| **Why does it exist?** | Because in complex domains, misunderstanding the business costs more than any technical choice |
| **Where does it belong?** | At the centre, with infrastructure as a detail around it |
| **When should I use it?** | Complex, long-lived domains — its language and value objects are worth adopting everywhere |

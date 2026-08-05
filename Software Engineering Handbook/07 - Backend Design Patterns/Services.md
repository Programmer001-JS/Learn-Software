# Services

> **In one line —** the layer that holds your business rules, knows nothing about HTTP or SQL, and is therefore the only part of the system you can test cheaply.

| | |
|---|---|
| **Category** | Architectural Pattern |
| **Architectural Layer** | Application / Domain |
| **Also called** | Application service · Use case · Interactor · Action |
| **Related notes** | [Controllers](Controllers.md) · [Repository Pattern](Repository%20Pattern.md) · [Clean Architecture](Clean%20Architecture.md) · [Domain Driven Design](Domain%20Driven%20Design.md) · [Transactions](../08%20-%20Databases%20and%20Data/Transactions.md) |

---

## 1. Short Definition

*What is it?*

A service contains the business logic for one use case: what must happen, in what order, under which rules. It is called by controllers, jobs and CLI commands alike, and it knows nothing about how it was invoked.

---

## 2. Purpose

*What is its main purpose?*

To give business rules a home that is independent of delivery mechanism and storage — so the rules can be read, tested and changed without touching HTTP or SQL.

---

## 3. Problem

*What engineering problem does it solve?*

Without a service layer, business rules end up in one of two bad places:

```text
IN THE CONTROLLER               IN THE MODEL / ORM ENTITY
tied to HTTP                    tied to the database schema
untestable without a server     entities grow to thousands of lines
unusable from a job             every rule loads the whole object graph
duplicated in every endpoint    circular dependencies between entities
   that needs the same rule
```

---

## 4. Architecture Position

```text
Controller (HTTP)   ·   CLI command   ·   Queue consumer   ·   Scheduled job
        └──────────────────┬──────────────────┘
                           ↓
                       SERVICE          ← business rules live here
                           ↓
                      Repository
                           ↓
                       Database
```

> [!IMPORTANT]
> The point of the service layer is visible in that diagram: **four different entry points, one implementation of the rules.** If a rule can only be reached through an HTTP request, it will eventually be reimplemented — differently — somewhere else.

---

## 5. What a service does

```python
class OrderService:
    def __init__(self, orders: OrderRepo, stock: StockRepo, events: EventBus):
        self.orders, self.stock, self.events = orders, stock, events

    async def create(self, user: User, cmd: CreateOrder) -> Order:
        available = await self.stock.available(cmd.sku)
        if available < cmd.quantity:
            raise OutOfStockError(cmd.sku)          # domain error, not HTTP

        price = self.pricing.for_user(user, cmd)     # business rule
        order = Order.new(user.id, cmd, price)

        async with self.uow:                          # transaction boundary
            await self.orders.add(order)
            await self.stock.reserve(cmd.sku, cmd.quantity)

        await self.events.publish(OrderCreated(order.id))   # after commit
        return order
```

Note what is absent: no `request`, no status codes, no SQL, no JSON.

---

## 6. Service or domain model?

Two schools, and both are defensible:

```text
ANEMIC + THICK SERVICE            RICH DOMAIN MODEL
entities are data holders         entities enforce their own invariants
all rules live in services        services orchestrate, entities decide
simpler, more procedural          harder to learn, scales better in complex domains
fine for CRUD-heavy systems       see Domain Driven Design
```

> [!TIP]
> Start with services holding the rules. Move rules into entities when you notice the same invariant being checked in several services — that repetition is the signal the rule belongs to the object itself.

---

## 7. Transaction boundaries belong here

> [!IMPORTANT]
> The service is the natural **transaction boundary**. A controller does not know which operations must succeed together; a repository is too fine-grained. One use case, one transaction.

```text
Controller     no transaction knowledge
    ↓
SERVICE        begin → ... → commit / rollback     ← the boundary
    ↓
Repository     participates in the ambient transaction
```

See [Transactions](../08%20-%20Databases%20and%20Data/Transactions.md).

---

## 8. Real World Example

- **Spring `@Service`** with `@Transactional` is this pattern made explicit by the framework.
- **NestJS providers** — services are the default unit of business logic.
- **Laravel Actions** and Django "service modules" are community responses to fat controllers and fat models.
- **The clearest real-world proof**: an operation available both as an API endpoint and as a nightly batch job, sharing one service.

---

## 9. Communication and Dependencies

- **[Controllers](Controllers.md)**, jobs and consumers call it
- **[Repositories](Repository%20Pattern.md)** are injected into it — never concrete database code
- **Other services** may be composed, carefully
- **An event bus** for things that happen after the fact

> [!CAUTION]
> Services calling services calling services becomes a call graph nobody can follow. If service A always needs service B, consider whether they are really one use case, or whether B should be a domain object instead.

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Introduce a service layer as soon as there are genuine business rules — anything beyond "insert this row".

> [!CAUTION]
> For pure CRUD with no rules, a service that only forwards calls to a repository is pass-through noise. Be honest about it: a thin controller calling a repository directly is better than a layer that adds nothing but a file.

---

## 11. Advantages and Disadvantages

**Advantages**
- Business rules testable without HTTP, database or framework
- Reusable from every entry point
- One clear transaction boundary
- Rules are readable in one place, in domain language
- Storage and delivery can change without touching them

**Disadvantages**
- More files and indirection
- Easy to build an anemic layer that only forwards calls
- Service-to-service dependencies can become tangled
- Over-applied to simple CRUD, it is ceremony

---

## 12. Performance Impact

Structural, not performance-related. The one genuine risk is **hidden N+1 queries**: a service calling a repository inside a loop looks innocent and produces hundreds of queries. Load collections in one call and operate on them in memory.

---

## 13. Security Considerations

> [!IMPORTANT]
> **Authorisation belongs in or above the service, never only in the controller.** If the rule "you may only cancel your own order" lives in the HTTP layer, the queue consumer that also cancels orders has no such check.

- Services should raise **domain exceptions** (`PermissionDeniedError`), which the HTTP layer maps to status codes
- **Validate business invariants here** — validation checked the shape of the input, not whether the operation makes sense
- **Audit logging** belongs here: the service knows *what happened*, the controller only knows a request arrived
- Beware **race conditions**: "check stock, then reserve" needs a transaction or a database-level constraint, not two separate queries

---

## 14. Mental Model

> [!NOTE]
> **The service layer is the company's rulebook.**
>
> The receptionist ([controller](Controllers.md)) takes requests, the archive ([repository](Repository%20Pattern.md)) stores records — and the rulebook says what is actually allowed to happen. It does not care whether the request arrived by phone, email or in person, which is exactly why it can be applied consistently to all three.

---

## 15. Mini Architecture Diagram

```text
HTTP   CLI   Queue   Cron
  └──────┬─────┬──────┘
         ↓
┌────── SERVICE ──────┐
│  authorisation      │
│  business rules     │
│  transaction        │
│  orchestration      │
│  domain events      │
└──────────┬──────────┘
           ↓
      Repository
           ↓
       Database
```

---

## 16. Complete Request Flow

```text
Controller calls order_service.create(user, cmd)
    ↓
AUTHORISE: may this user create an order for this account?
    ↓
Load what is needed via repositories
    ↓
Apply BUSINESS RULES: stock, pricing, discounts, limits
    ↓
Rule violated → raise OutOfStockError (domain, not HTTP)
    ↓
BEGIN TRANSACTION
    persist the order
    reserve stock
COMMIT
    ↓
Publish OrderCreated — after commit, so no event fires for a rolled-back order
    ↓
Return the domain object; the controller maps it to a DTO
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> Services hold business rules independently of HTTP and SQL — which is what makes those rules testable, reusable from every entry point, and enforced consistently.

---

## 18. Common Mistakes

- **Passing the request object into the service**, coupling it to HTTP
- **Raising HTTP exceptions** from services
- **Authorisation only in the controller**, so other entry points bypass it
- **Anemic services** that only forward to repositories
- **N+1 queries** from repository calls inside loops
- **No clear transaction boundary**, so partial writes survive failures
- **Publishing events before commit**, announcing things that never happened

---

## 19. Open Source Technologies

- **Spring `@Service` / `@Transactional`**
- **NestJS providers**
- **Laravel Actions**, **Django service modules**
- **MediatR** (.NET) — one handler per use case
- **pytest**, **xUnit**, **Jest** — where the payoff appears: fast tests with no HTTP or database

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Pick one business rule in your system and check whether a background job could bypass it.
- [ ] Find one service that takes a request or response object as a parameter, and remove that dependency.
- [ ] Write a unit test for one service with no HTTP and no database, using fake repositories.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Entry points (HTTP, CLI, queue, cron)
    ↓
Service  (rules, authorisation, transaction)
    ↓
Repository → Database
```

## 2. Request Flow

```text
Input       a typed command plus an authenticated actor
    ↓
Processing  authorise → apply rules → transaction → persist → publish events
    ↓
Output      a domain object, or a domain exception
```

## 3. Real-World Usage

**Spring's `@Service` and `@Transactional`** made this pattern explicit in the framework itself, and the transaction boundary sitting on the service method is the clearest expression of why the layer exists.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The layer holding business rules for a use case |
| **Why does it exist?** | So rules are independent of delivery mechanism and storage |
| **Where does it belong?** | Between controllers and repositories |
| **When should I use it?** | Whenever there are real rules — not for pure pass-through CRUD |

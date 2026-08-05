# Clean Architecture

> **In one line —** put business rules at the centre and make every dependency point inward, so the database and the web framework become replaceable details.

| | |
|---|---|
| **Category** | Architecture Pattern |
| **Architectural Layer** | Whole application |
| **Also called** | Hexagonal · Ports and Adapters · Onion |
| **Related notes** | [SOLID Principles](SOLID%20Principles.md) · [Repository Pattern](Repository%20Pattern.md) · [Services](Services.md) · [Domain Driven Design](Domain%20Driven%20Design.md) · [Dependency Injection](Dependency%20Injection.md) |

---

## 1. Short Definition

*What is it?*

Clean Architecture organises code into concentric layers where **all dependencies point inward**. Business rules sit at the centre and know nothing about HTTP, SQL, or any framework.

---

## 2. Purpose

*What is its main purpose?*

To make the parts most likely to change — the web framework, the database, the message broker — into peripheral details that can be replaced without touching the rules that define the business.

---

## 3. Problem

*What engineering problem does it solve?*

In a conventional layered application, business logic depends on the ORM, which depends on the database. Change the database and everything above it moves.

```text
CONVENTIONAL                          CLEAN
Controller → Service → ORM → DB       Controller → Use Case → Interface
                                                                  ▲
   business logic depends on                              Repository implements
   the framework and the database                                  ↓
                                                                   DB
                                       business logic depends on nothing
```

---

## 4. The layers

```text
        ┌──────────────────────────────────────┐
        │  Frameworks & Drivers                │  web, DB, UI, external APIs
        │  ┌────────────────────────────────┐  │
        │  │  Interface Adapters            │  │  controllers, presenters,
        │  │  ┌──────────────────────────┐  │  │  repository implementations
        │  │  │  Use Cases               │  │  │  application-specific rules
        │  │  │  ┌────────────────────┐  │  │  │
        │  │  │  │  Entities          │  │  │  │  enterprise business rules
        │  │  │  └────────────────────┘  │  │  │
        │  │  └──────────────────────────┘  │  │
        │  └────────────────────────────────┘  │
        └──────────────────────────────────────┘

                 dependencies point INWARD only
```

| Layer | Contains | Knows about |
|---|---|---|
| **Entities** | Core business objects and invariants | Nothing |
| **Use Cases** | Application operations | Entities only |
| **Interface Adapters** | Controllers, repositories, presenters | Use cases |
| **Frameworks & Drivers** | Web framework, ORM, database, queues | Everything outward of it |

---

## 5. The dependency rule

> [!IMPORTANT]
> **Nothing in an inner circle may know anything about an outer circle.** No entity imports a framework. No use case imports SQLAlchemy. The database is a plugin to your application, not its foundation.

The mechanism is [dependency inversion](SOLID%20Principles.md): the use case *defines* the interface it needs, and the outer layer implements it.

```python
# inner layer — defines what it needs
class OrderRepository(Protocol):
    async def add(self, order: Order) -> None: ...

# outer layer — implements it
class SqlOrderRepository(OrderRepository):
    ...   # SQLAlchemy lives here
```

---

## 6. Ports and adapters

The same idea under a clearer name:

```text
        HTTP adapter ─┐                    ┌─ PostgreSQL adapter
        CLI adapter  ─┤                    ├─ Redis adapter
        Queue adapter ┤→ [ PORTS ] APP [ PORTS ] ├─ Email adapter
        Cron adapter ─┘                    └─ S3 adapter

        driving side                        driven side
        (things that call you)              (things you call)
```

A **port** is an interface; an **adapter** is an implementation. The application does not know whether a request arrived over HTTP or from a cron job.

---

## 7. Real World Example

- **Testing** is the clearest payoff: the entire application logic runs with in-memory adapters, no database, no HTTP, in milliseconds.
- **Replacing infrastructure** — moving from REST to gRPC, or Postgres to DynamoDB, touches only adapters.
- **Long-lived enterprise systems** — where the business rules outlive several generations of framework.
- **[Domain Driven Design](Domain%20Driven%20Design.md)** is usually implemented on this structure.

---

## 8. What it costs

> [!CAUTION]
> This is the most over-applied architecture in this handbook. It is genuinely expensive:
>
> - Many more files and interfaces
> - Mapping between layers — domain objects, DTOs, ORM entities, request models
> - A simple CRUD endpoint may touch six files
> - Steep onboarding for anyone who has not seen it
> - The database independence you paid for is almost never exercised

```text
Is the domain complex, with real rules that outlive the framework?
    ↓ YES                                  ↓ NO
Clean Architecture earns its cost      A layered controller → service →
                                       repository structure is enough,
                                       and honest
```

---

## 9. When To Use / When NOT To Use

> [!TIP]
> Use it for systems with substantial domain complexity, long expected lifetimes, several delivery mechanisms, or where the business rules genuinely are the product.

> [!CAUTION]
> Do not use it for CRUD applications, prototypes, small services, or anywhere the ORM model *is* the domain model. Building it "in case we need it" is the textbook example of paying for optionality you will never exercise.

---

## 10. Advantages and Disadvantages

**Advantages**
- Business rules testable with no infrastructure at all
- Framework and database are replaceable
- Multiple delivery mechanisms over one core
- Rules readable in domain language, uncluttered by plumbing
- Survives framework generations

**Disadvantages**
- Substantial boilerplate and mapping code
- Steep learning curve
- Over-engineering for most applications
- The flexibility is frequently never used
- Debugging spans more layers

---

## 11. Performance Impact

| Aspect | Impact |
|---|---|
| **Runtime** | Negligible — a few extra function calls |
| **Mapping** | Object-to-object conversion, measurable only in very hot paths |
| **The real risk** | Repository interfaces so generic that callers fetch too much and filter in memory |

---

## 12. Security Considerations

> [!IMPORTANT]
> Clean Architecture is genuinely good for security when done well: **authorisation lives in the use case**, so every delivery mechanism — HTTP, CLI, queue consumer — is subject to the same rules. There is no path that bypasses them.

- **Domain invariants are enforced in entities**, so no adapter can construct an invalid object
- **But**: deep layering can obscure where checks happen. If nobody can answer "where is this authorised?", the architecture has failed at its own goal
- **Do not let DTO mapping become an accidental data leak** — map explicitly, never automatically copy every field outward

---

## 13. Mental Model

> [!NOTE]
> **Clean Architecture is a building where the plumbing is on the outside.**
>
> The rooms — where people actually live — do not depend on which pipes are installed. Replace the boiler without touching the bedrooms. The cost is that the building is more complicated to construct, and a garden shed does not need it.

---

## 14. Mini Architecture Diagram

```text
        HTTP        CLI        Queue        Cron
          └──────────┴───────────┴───────────┘
                          ↓
                    Controllers  (adapters)
                          ↓
                     USE CASES         ← application rules, authorisation
                          ↓
                      ENTITIES         ← invariants, no dependencies at all
                          ↑
                  Repository interfaces  (ports, defined by the inside)
                          ↑
        SQL adapter · Redis adapter · S3 adapter · Email adapter
```

---

## 15. Complete Request Flow

```text
HTTP POST /orders
    ↓
Controller (adapter): parse and validate → a request model
    ↓
Calls CreateOrderUseCase — knows nothing about HTTP
    ↓
Use case AUTHORISES the action
    ↓
Loads via OrderRepository INTERFACE (not the implementation)
    ↓
Entity enforces its own invariants — an invalid Order cannot exist
    ↓
Use case commits through a Unit of Work
    ↓
Returns a domain object
    ↓
Controller maps it to a response DTO
    ↓
The same use case, called by a cron job, applies identical rules
```

---

## 16. Key Takeaway

> [!IMPORTANT]
> Clean Architecture makes dependencies point inward so business rules survive changes of framework and database — and its cost is real enough that most applications should not pay it.

---

## 17. Common Mistakes

- **Applying it to a CRUD application** and calling the boilerplate rigour
- **Leaking ORM entities inward**, which breaks the dependency rule entirely
- **Framework imports in the domain layer** — the single clearest violation
- **Mapping layers that copy every field automatically**, defeating the point
- **Interfaces with one implementation** that will never have a second
- **Deep layering that hides where authorisation happens**
- **Confusing it with folder naming** — a `domain/` folder importing Django is not clean architecture

---

## 18. Open Source Technologies

- **Python `Protocol`**, **Go interfaces**, **TypeScript interfaces** — defining ports
- **ArchUnit**, **import-linter**, **dependency-cruiser** — enforce the dependency rule in CI
- **MediatR** (.NET) — one handler per use case
- Robert C. Martin, *Clean Architecture*; Alistair Cockburn's original hexagonal architecture writing

---

## 19. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 20. Workbook Exercise

- [ ] Check whether any file in your business logic imports your web framework or ORM.
- [ ] Take one use case and ask whether a CLI command could invoke it unchanged.
- [ ] Decide honestly whether your project's domain complexity justifies this architecture, and write down why.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Adapters (HTTP, CLI, queue)
    ↓
Use cases
    ↓
Entities
    ↑
Repository interfaces ← implemented by infrastructure adapters
```

## 2. Request Flow

```text
Input       a request from any delivery mechanism
    ↓
Processing  adapter → use case (authorise, apply rules) → entities → ports
    ↓
Output      a domain result, mapped outward to whatever the caller needs
```

## 3. Real-World Usage

The most convincing demonstration is **the test suite**: an application core that runs end to end with in-memory adapters, no database and no HTTP server, in milliseconds. Teams that achieve that find it changes how often they refactor.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An architecture where all dependencies point inward toward business rules |
| **Why does it exist?** | So frameworks and databases become replaceable details |
| **Where does it belong?** | As the overall structure of an application |
| **When should I use it?** | Complex, long-lived domains — not CRUD applications or prototypes |

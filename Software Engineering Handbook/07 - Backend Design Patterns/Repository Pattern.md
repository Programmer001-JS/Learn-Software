# Repository Pattern

> **In one line —** a collection-like interface over your data, so business logic asks for objects instead of writing queries.

| | |
|---|---|
| **Category** | Design Pattern |
| **Architectural Layer** | Data access |
| **Related notes** | [Services](Services.md) · [ORM](../08%20-%20Databases%20and%20Data/ORM.md) · [Clean Architecture](Clean%20Architecture.md) · [Dependency Injection](Dependency%20Injection.md) · [Transactions](../08%20-%20Databases%20and%20Data/Transactions.md) |

---

## 1. Short Definition

*What is it?*

A repository is an object that looks like an in-memory collection of domain objects — `add`, `get`, `find_by`, `remove` — and hides how they are actually stored.

---

## 2. Purpose

*What is its main purpose?*

To keep query syntax out of business logic. A service should say *what* it needs, not *how* to fetch it.

---

## 3. Problem

*What engineering problem does it solve?*

```text
WITHOUT A REPOSITORY                  WITH A REPOSITORY

# in the service                      # in the service
session.query(Order)                  orders = order_repo.pending_for(user.id)
  .join(Item)
  .filter(Order.user_id == user.id)   # in the repository
  .filter(Order.status == 'pending')  def pending_for(self, user_id):
  .options(joinedload(Order.items))       return (self.session.query(Order)
  .all()                                      .options(joinedload(Order.items))
                                              .filter_by(user_id=user_id,
↑ ORM syntax scattered through                          status='pending')
  every service; the same query               .all())
  written slightly differently in
  four places, one without eager      ↑ one place; fix the N+1 once
  loading
```

---

## 4. Architecture Position

```text
Controller
    ↓
Service          ← depends on a repository INTERFACE
    ↓
━━━ interface boundary ━━━
    ↓
Repository       ← the implementation, which knows SQL/ORM
    ↓
Database
```

> [!IMPORTANT]
> The value comes from the **direction of the dependency**. The service depends on an interface it defines; the repository implements it. Storage becomes a detail the domain does not know about — this is the core idea of [Clean Architecture](Clean%20Architecture.md).

---

## 5. What it looks like

```python
class OrderRepository(Protocol):
    async def get(self, id: OrderId) -> Order | None: ...
    async def add(self, order: Order) -> None: ...
    async def pending_for(self, user_id: UserId) -> list[Order]: ...


class SqlOrderRepository(OrderRepository):
    def __init__(self, session): self.session = session
    async def get(self, id): ...        # SQLAlchemy lives here, and only here
```

The service is constructed with `OrderRepository`; in tests it receives an in-memory implementation and never touches a database.

---

## 6. The honest objection

> [!CAUTION]
> **An ORM is already a repository.** Django's `Order.objects.filter(...)` and SQLAlchemy's session are abstractions over SQL. Wrapping them in another layer can be pure ceremony.

The pattern earns its place when:

- Business logic must be testable **without** a database
- Queries are complex and are reused across several services
- You genuinely might change storage — SQL to a document store, or add a cache in front
- The domain model differs from the database schema
- You want one place to fix N+1 problems

It does **not** earn its place in a small CRUD application where the ORM model is the domain model. Be honest about which you have.

---

## 7. Real World Example

- **Spring Data JPA** generates implementations from interface method names — the pattern as a framework feature.
- **[Clean Architecture](Clean%20Architecture.md) and hexagonal architecture** treat repositories as "ports" with database "adapters".
- **Testing** is where it pays: an in-memory repository makes service tests run in milliseconds instead of seconds.
- **Adding a cache** becomes a decorator around the repository, invisible to every caller.

---

## 8. Communication and Dependencies

- **[Services](Services.md)** depend on the interface
- **The ORM or database driver** is used inside the implementation
- **[Dependency injection](Dependency%20Injection.md)** supplies the concrete instance
- **Unit of Work** coordinates several repositories inside one [transaction](../08%20-%20Databases%20and%20Data/Transactions.md)

---

## 9. Repository vs DAO vs Active Record

```text
ACTIVE RECORD     order.save()          the object saves itself
                  Django, Eloquent, ActiveRecord — simple, couples domain to storage

DAO               orderDao.insert(row)  table-oriented, close to SQL

REPOSITORY        repo.add(order)       collection-oriented, domain-shaped,
                                        returns domain objects
```

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Use it when the domain is complex enough to deserve protection from storage details, and when fast tests without a database are valuable.

> [!CAUTION]
> Do not add it reflexively. A repository whose every method is a one-line pass-through to the ORM has added a file and removed nothing. And **do not build a generic `Repository<T>`** with `findAll`, `findById`, `save` — a generic interface is just the ORM with extra steps, and it encourages loading everything and filtering in memory.

---

## 11. Advantages and Disadvantages

**Advantages**
- Query logic in one place, reused and optimised once
- Business logic testable with no database
- Storage technology becomes replaceable
- A natural place to add caching, retries or read replicas
- Domain code reads in domain language

**Disadvantages**
- Another layer for something the ORM partly does already
- Leaky in practice — pagination, transactions and eager loading tend to bleed through the interface
- Generic repositories are actively harmful
- Real ceremony in simple applications

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Overhead** | Negligible |
| **N+1 queries** | **Fixed in one place** — the strongest practical argument for the pattern |
| **Eager loading** | Encapsulated, so callers cannot forget it |
| **Risk** | A naive `find_all()` that callers filter in memory |

---

## 13. Security Considerations

> [!CAUTION]
> **A repository must never build SQL by string concatenation.** Parameterised queries are the defence against SQL injection, and a repository is where raw SQL is most likely to appear — precisely because it is the layer allowed to write it.

- **Multi-tenancy** — a repository that always filters by tenant is far safer than trusting every caller to remember; this is a genuinely strong reason to use the pattern
- **Soft deletes** — enforce the "not deleted" filter in one place, or deleted records will eventually leak
- **Do not expose raw query builders** from the repository, or the encapsulation is gone
- **Authorisation stays in the [service](Services.md)** — a repository fetches, it does not decide who may see

---

## 14. Mental Model

> [!NOTE]
> **A repository is a librarian.**
>
> You ask for "everything published by this author since 2020". You do not care whether they walk to a shelf, search a microfiche or query a database. You get books. And if the library reorganises entirely, your request is unchanged.

---

## 15. Mini Architecture Diagram

```text
Service
    ↓  depends on
OrderRepository  (interface, defined by the domain)
    ↑  implemented by
┌────────────┬─────────────────┬──────────────┐
SqlOrderRepo  CachedOrderRepo   InMemoryRepo
    ↓              ↓                 ↓
PostgreSQL      Redis + DB        tests
```

---

## 16. Complete Request Flow

```text
Controller → OrderService.create()
    ↓
Service calls order_repo.pending_for(user.id)
    ↓
Repository builds the query, with eager loading already applied
    ↓
Parameterised SQL sent to PostgreSQL
    ↓
Rows mapped to domain Order objects
    ↓
Service applies business rules to real objects, not rows
    ↓
Service calls order_repo.add(order) inside a transaction
    ↓
In tests: the same service, given InMemoryRepo, runs with no database at all
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> A repository hides how data is stored so business logic can ask for objects — worth it when the domain is complex or tests must be fast, and ceremony when the ORM model already is the domain model.

---

## 18. Common Mistakes

- **Generic `Repository<T>`** interfaces that add nothing
- **Leaking the ORM** by returning query builders or ORM entities
- **`find_all()` then filtering in memory** — a full table scan dressed as clean code
- **Business logic inside the repository** — it fetches, it does not decide
- **A repository per table** instead of per aggregate
- **Adding it to a small CRUD app** and calling the ceremony architecture
- **Raw string SQL** inside repository methods

---

## 19. Open Source Technologies

- **Spring Data JPA** — repositories generated from interfaces
- **SQLAlchemy**, **Prisma**, **Entity Framework**, **Drizzle** — what sits inside the implementation
- **Testcontainers** — real databases in integration tests when you do want them
- **Unit of Work** implementations for transaction coordination

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Find the same query written in two different places in your codebase and unify it behind one method.
- [ ] Write one service test using an in-memory repository, with no database.
- [ ] Decide honestly whether your project needs this pattern — and write down the reason either way.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Service → Repository interface → implementation → ORM → Database
```

## 2. Request Flow

```text
Input       a domain-language request ("pending orders for this user")
    ↓
Processing  translated to a parameterised query with correct eager loading
    ↓
Output      domain objects, not rows
```

## 3. Real-World Usage

**Spring Data JPA** generates a working repository from an interface declaration alone. That a major framework chose to make this pattern automatic is a reasonable indication of how often it is worth having.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A collection-like abstraction over data storage |
| **Why does it exist?** | To keep query syntax out of business logic |
| **Where does it belong?** | Between services and the ORM |
| **When should I use it?** | Complex domains and fast tests — not simple CRUD |

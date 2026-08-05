# ORM

> **In one line —** a translator between database rows and objects; it removes most SQL boilerplate and hides exactly the details that cause performance problems.

| | |
|---|---|
| **Full name** | Object-Relational Mapping |
| **Category** | Data Access Pattern |
| **Architectural Layer** | Between application and database |
| **Related notes** | [SQL](SQL.md) · [Repository Pattern](../07%20-%20Backend%20Design%20Patterns/Repository%20Pattern.md) · [SQLAlchemy](SQLAlchemy.md) · [Prisma](Prisma.md) · [Hibernate](Hibernate.md) · [Entity Framework](Entity%20Framework.md) |

---

## 1. Short Definition

*What is it?*

An ORM maps database tables to classes and rows to objects, generating SQL from method calls so you work with your language's types instead of writing queries.

---

## 2. Purpose

*What is its main purpose?*

To remove the repetitive, error-prone glue between a relational schema and an object-oriented application — and to parameterise queries safely by default.

---

## 3. Problem

*What engineering problem does it solve?*

```text
WITHOUT AN ORM                        WITH AN ORM
cursor.execute("SELECT id, name, ...  user = User.get(42)
                FROM users WHERE id=%s", (42,))
row = cursor.fetchone()               ↑ typed object, relationships
user = User(row[0], row[1], ...)        navigable, SQL parameterised
    ↓                                   automatically
manual mapping in every query
column order coupling
easy to forget parameterisation
```

---

## 4. Architecture Position

```text
Service layer
    ↓
Repository  (optional)
    ↓
ORM              ← maps objects ↔ rows, generates SQL
    ↓
Database driver
    ↓
Database
```

---

## 5. The two mapping styles

```text
ACTIVE RECORD                         DATA MAPPER
the object knows how to save itself   a separate layer moves data
user.save()                           session.add(user); session.commit()
    ↓                                     ↓
Django ORM, Eloquent, ActiveRecord    SQLAlchemy, Hibernate, Doctrine
simple, fast to write                 domain objects stay storage-agnostic
domain coupled to persistence         more ceremony, cleaner separation
```

---

## 6. The N+1 problem — the defining ORM failure

> [!CAUTION]
> This single issue causes more ORM-related performance incidents than everything else combined, and it is invisible in the source code.

```python
orders = Order.objects.all()          # 1 query
for order in orders:
    print(order.customer.name)        # ← one query PER ORDER
```

```text
100 orders  →  101 queries  →  ~2 seconds
    ↓  fix
Order.objects.select_related('customer')   →  1 query  →  ~20 ms
```

| ORM | Fix |
|---|---|
| Django | `select_related` (join) / `prefetch_related` (second query) |
| SQLAlchemy | `joinedload` / `selectinload` |
| Hibernate | `JOIN FETCH` / entity graphs |
| Prisma | `include` |
| Entity Framework | `Include` |

> [!TIP]
> Install a query counter in development — django-debug-toolbar, Laravel Debugbar, SQLAlchemy echo. A page that runs 200 queries is obvious the moment you can see the number, and invisible until then.

---

## 7. Lazy loading — the mechanism behind it

```text
order = get_order(42)              ← one query
order.customer                     ← triggers ANOTHER query, silently
order.items                        ← and another
```

Convenient when reading one object. Catastrophic inside a loop, and the reason lazy loading is disabled by default in Entity Framework Core and discouraged in modern Hibernate practice.

---

## 8. Migrations

The genuinely valuable feature people underrate:

```text
Change the model
    ↓
Generate a migration     (a versioned, reviewable schema change)
    ↓
Apply it in every environment, in order
    ↓
Schema and code stay in sync, and the change is in version control
```

> [!CAUTION]
> **Always read generated migrations before applying them.** Auto-generated DDL has dropped columns, rebuilt large tables and locked production for minutes. On a big table, an innocuous-looking `ALTER` can be an outage.

---

## 9. Real World Example

- **Django ORM and Rails ActiveRecord** made rapid CRUD development the norm.
- **Hibernate** dominated enterprise Java and taught a generation about lazy loading the hard way.
- **Prisma** brought type-safe generated clients to TypeScript.
- **Most teams use a hybrid**: the ORM for CRUD, raw SQL for reports and complex aggregations.

---

## 10. Communication and Dependencies

- **A database driver** underneath
- **A connection pool** — usually managed by the ORM
- **Migration tooling** — Alembic, Django migrations, Flyway, EF migrations
- **The [repository pattern](../07%20-%20Backend%20Design%20Patterns/Repository%20Pattern.md)**, optionally, above it

---

## 11. Alternatives

```text
Full ORM         Django, Hibernate, SQLAlchemy ORM — most abstraction
    ↓
Query builder    Knex, jOOQ, SQLAlchemy Core — composable, SQL-shaped
    ↓
Micro-ORM        Dapper, asyncpg + mapping — you write SQL, it maps results
    ↓
Raw SQL          full control, full responsibility
```

> [!TIP]
> **Query builders are underrated.** They give composability and parameterisation without hiding the query shape — often the best balance for read-heavy applications.

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use an ORM for standard CRUD, where it removes real boilerplate and enforces parameterised queries. Use it for the 80% of queries that are simple.

> [!CAUTION]
> Drop to SQL for reports, complex aggregations, window functions, bulk operations and anything performance-critical. An ORM generating a five-table aggregation is usually both slower and harder to read than the SQL it replaced.

---

## 13. Advantages and Disadvantages

**Advantages**
- Far less boilerplate for common operations
- SQL injection prevented by default
- Migrations version-control the schema
- Database portability, in principle
- Relationships navigable as object properties

**Disadvantages**
- **N+1 queries**, hidden by the abstraction
- Generated SQL can be poor for complex queries
- A real learning curve — the ORM is a second system to understand
- Encourages ignorance of what the database actually does
- Bulk operations are frequently slow unless bypassed

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Mapping overhead** | Small |
| **N+1 queries** | **The dominant risk** — orders of magnitude |
| **Bulk insert/update** | Often far slower than a single SQL statement |
| **`SELECT *` by default** | Most ORMs fetch every column unless told otherwise |
| **Identity map** | A useful per-request cache in some ORMs |

---

## 15. Security Considerations

> [!IMPORTANT]
> **ORMs parameterise queries by default, which prevents SQL injection.** This is one of their strongest arguments — more so than the convenience.

> [!CAUTION]
> The protection ends the moment you build SQL as a string:
> ```python
> User.objects.raw(f"SELECT * FROM users WHERE email = '{email}'")   # ✗ injectable
> User.objects.raw("SELECT * FROM users WHERE email = %s", [email])  # ✓ safe
> ```

- **Mass assignment** — binding a request body directly to a model lets a caller set `is_admin`. Use an explicit input schema
- **Never return ORM entities from an API** — they serialise every column, including ones added later
- **Least privilege** — the application's database user should not hold DDL rights, even though migrations do

---

## 16. Mental Model

> [!NOTE]
> **An ORM is an interpreter between you and someone who only speaks SQL.**
>
> For everyday conversation it is faster and you make fewer mistakes. For a legal negotiation you want to read the actual wording — because the interpreter may have phrased it in a way that is technically correct and disastrously inefficient.

---

## 17. Mini Architecture Diagram

```text
Application objects
    ↕
ORM  (mapping · relationships · unit of work · migrations)
    ↕
Generated SQL
    ↕
Driver → connection pool
    ↕
Database rows
```

---

## 18. Complete Request Flow

```text
service.get_recent_orders(user)
    ↓
ORM builds a query from the model definitions
    ↓
Relationships eager-loaded?  → 1 query
Lazy?                        → 1 + N queries          ← the trap
    ↓
Parameterised SQL sent to the database
    ↓
Rows returned, mapped into objects, relationships wired up
    ↓
Identity map: the same row requested twice returns the same object
    ↓
Changes tracked; commit flushes them as UPDATE statements
    ↓
Transaction committed
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> An ORM removes boilerplate and prevents SQL injection, but hides query behaviour — so watch the query count, and drop to SQL wherever the query is the point.

---

## 20. Common Mistakes

- **N+1 queries** — the near-universal ORM performance bug
- **Never looking at the generated SQL**
- **Bulk operations through the ORM** row by row
- **Binding request bodies directly to models** — mass assignment
- **Returning entities from API endpoints**
- **Applying migrations without reading them**
- **Fighting the ORM** on a complex query instead of writing SQL
- **Treating it as a reason not to learn SQL** — the opposite is true

---

## 21. Open Source Technologies

- **[SQLAlchemy](SQLAlchemy.md)**, **Django ORM** — Python
- **[Prisma](Prisma.md)**, **Drizzle**, **TypeORM** — TypeScript
- **[Hibernate](Hibernate.md)**, **jOOQ** — JVM
- **[Entity Framework Core](Entity%20Framework.md)**, **Dapper** — .NET
- **Eloquent**, **Doctrine** — PHP
- **Alembic**, **Flyway**, **Liquibase** — migrations

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Enable query logging and find the page in your application that runs the most queries.
- [ ] Fix one N+1 with eager loading and measure the difference.
- [ ] Take one ORM query and read the SQL it generates. Would you have written that?

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Objects ↕ ORM ↕ SQL ↕ Database
```

## 2. Request Flow

```text
Input       a method call on a model or query object
    ↓
Processing  translated to parameterised SQL, executed, mapped back to objects
    ↓
Output      typed objects — and a query count you should be watching
```

## 3. Real-World Usage

**Hibernate** taught the Java world about lazy loading through years of production incidents, which is why modern ORMs default to eager or explicit loading. The N+1 problem is the single most transferable piece of ORM knowledge there is.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A mapping layer between database rows and application objects |
| **Why does it exist?** | To remove boilerplate and make parameterised queries the default |
| **Where does it belong?** | Between your service layer and the database driver |
| **When should I use it?** | For routine CRUD — with raw SQL for reports and hot paths |

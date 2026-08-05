# SQLAlchemy

> **In one line —** Python's most capable database toolkit: a full ORM sitting on top of a query builder that never hides SQL from you.

| | |
|---|---|
| **Category** | ORM / Database Toolkit |
| **Architectural Layer** | Data access |
| **Language** | Python |
| **Style** | Data Mapper (with an optional Core layer) |
| **Related notes** | [ORM](ORM.md) · [SQL](SQL.md) · [PostgreSQL](PostgreSQL.md) · [FastAPI](../06%20-%20Backend%20Architecture/FastAPI.md) · [Repository Pattern](../07%20-%20Backend%20Design%20Patterns/Repository%20Pattern.md) |

---

## 1. Short Definition

*What is it?*

SQLAlchemy is a Python database toolkit in two layers: **Core**, a SQL expression builder, and **ORM**, an object mapper built on top of it. You can use either, or both, in the same application.

---

## 2. Purpose

*What is its main purpose?*

To give Python full access to SQL's power while removing the boilerplate — without pretending the database is not there.

---

## 3. Problem

*What engineering problem does it solve?*

Most ORMs force a choice: convenient object mapping, or control over the query. SQLAlchemy's design refuses that trade — the ORM is built *on* an expression language you can drop into at any point, in the same session and the same transaction.

```text
ORM layer      session.query(User).filter(...)      objects, relationships
    ↓  built on
CORE layer     select(users).where(...)             composable SQL expressions
    ↓
Actual SQL     you can always see it, and write it
```

---

## 4. Architecture Position

```text
FastAPI / Flask
    ↓
Repository (optional)
    ↓
SQLAlchemy ORM  — Session, identity map, unit of work
    ↓
SQLAlchemy Core — expression language
    ↓
DBAPI driver (psycopg, asyncpg)
    ↓
PostgreSQL
```

---

## 5. The Session — the concept to understand first

```text
The Session is:
    a TRANSACTION boundary
    an IDENTITY MAP        — the same row fetched twice is the same object
    a UNIT OF WORK         — tracks changes, flushes them on commit
```

```python
with Session(engine) as session:
    user = session.get(User, 42)
    user.name = "Ana"          # no SQL yet — the change is tracked
    session.commit()           # UPDATE issued here
```

> [!IMPORTANT]
> **A Session is not thread-safe and is not a global.** In a web application it must be scoped to one request and closed afterwards. A leaked session holds a database connection and an open transaction — which blocks PostgreSQL's autovacuum and, eventually, exhausts the pool.

---

## 6. Modern SQLAlchemy 2.0 style

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase): pass

class Order(Base):
    __tablename__ = "orders"
    id:      Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    total:   Mapped[Decimal]

stmt = select(Order).where(Order.user_id == 42).options(selectinload(Order.items))
orders = session.scalars(stmt).all()
```

> [!TIP]
> SQLAlchemy 2.0 unified the ORM and Core query syntax around `select()`, and added real type annotations. If you learned the 1.x `session.query()` style, the 2.0 style is worth relearning — it is what current documentation assumes.

---

## 7. Eager loading — the N+1 fix

```python
# ✗ N+1
for order in session.scalars(select(Order)):
    print(order.customer.name)         # one query per order

# ✓ one extra query total
select(Order).options(selectinload(Order.customer))

# ✓ single joined query
select(Order).options(joinedload(Order.customer))
```

| Strategy | Behaviour | Use for |
|---|---|---|
| `selectinload` | A second `IN (...)` query | **Collections** — the usual best choice |
| `joinedload` | A single `LEFT JOIN` | Many-to-one, small relations |
| `subqueryload` | A subquery | Legacy; usually superseded |
| `raiseload` | **Raises** on lazy access | Excellent in tests — makes N+1 fail loudly |

> [!TIP]
> `lazy="raise"` on relationships during development turns every accidental N+1 into an exception instead of a silent slowdown. It is the single most effective habit with this library.

---

## 8. Async support

```python
async with AsyncSession(engine) as session:
    result = await session.scalars(select(Order).where(...))
```

Async SQLAlchemy works well with [FastAPI](../06%20-%20Backend%20Architecture/FastAPI.md) and `asyncpg`.

> [!CAUTION]
> **Lazy loading does not work in async contexts** — accessing an unloaded relationship raises rather than silently issuing a query. This is a feature, not a limitation: it forces explicit eager loading, which is what you wanted anyway.

---

## 9. Real World Example

- **The standard data layer for FastAPI and Flask** applications.
- **Alembic**, its migration tool, is the de facto standard for Python schema migrations.
- **Airflow, Superset and many data tools** use it internally.
- **SQLModel** wraps SQLAlchemy plus Pydantic for FastAPI-shaped projects.

---

## 10. Communication and Dependencies

- **A DBAPI driver** — `psycopg` (sync), `asyncpg` (async)
- **Alembic** for migrations
- **A connection pool** — built in, configured on the engine
- **The database** — PostgreSQL, MySQL, SQLite and others

---

## 11. SQLAlchemy vs Django ORM

| | SQLAlchemy | Django ORM |
|---|---|---|
| **Style** | Data Mapper | Active Record |
| **Coupling** | Domain classes independent of storage | Models are the framework |
| **Power** | Higher — full SQL expression access | Simpler, less capable on complex queries |
| **Learning curve** | Steeper | Gentler |
| **Best with** | FastAPI, Flask, standalone services | Django |

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use SQLAlchemy for Python applications outside Django, especially where queries get complex or where you want the domain model separate from persistence.

> [!CAUTION]
> Inside Django, use Django's ORM — fighting the framework's integration is not worth it. And for a small script, `psycopg` with plain SQL is often clearer than setting up an engine, a session and a declarative base.

---

## 13. Advantages and Disadvantages

**Advantages**
- Genuinely powerful — matches SQL's expressiveness
- Core and ORM in one toolkit, mixable freely
- Explicit control over loading strategies
- Data Mapper keeps domain classes storage-agnostic
- Alembic migrations are excellent
- Mature async support

**Disadvantages**
- Steep learning curve — the Session model takes time
- Verbose compared with Django or Prisma
- Session lifecycle mistakes are a common source of production bugs
- The 1.x → 2.0 style change means much online material is outdated

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **N+1** | The main risk; fixed with explicit loading strategies |
| **Identity map** | Deduplicates objects within a session — a free per-request cache |
| **Bulk operations** | Use `insert()`/`update()` from Core; the ORM row-by-row path is slow |
| **Connection pool** | Configure `pool_size` and `max_overflow` against your database limit |
| **Change tracking** | A cost on very large object graphs |

---

## 15. Security Considerations

> [!IMPORTANT]
> SQLAlchemy parameterises everything by default — including values inside `text()` when you use bound parameters. Injection requires deliberately building a string.

```python
session.execute(text(f"SELECT * FROM users WHERE email = '{email}'"))  # ✗
session.execute(text("SELECT * FROM users WHERE email = :e"), {"e": email})  # ✓
```

- **Column and table names cannot be parameterised** — validate against an allow-list if they must be dynamic
- **Never return ORM models directly from an API** — use a Pydantic response model
- **Leaked sessions hold transactions open**, which is both a performance and an availability problem
- **Scope the session to the request** and always close it, ideally via a dependency

---

## 16. Mental Model

> [!NOTE]
> **SQLAlchemy is a translator who will also hand you the original document.**
>
> Most ORMs translate and hide the source. This one translates fluently, and the moment you need the exact wording, it gives you the original text and lets you edit it — in the same conversation.

---

## 17. Mini Architecture Diagram

```text
Application
    ↓
Session  (transaction · identity map · unit of work)
    ↓
ORM query  →  Core expression  →  SQL text
    ↓
Connection pool
    ↓
Database
```

---

## 18. Complete Request Flow

```text
Request → dependency creates a Session (scoped to this request)
    ↓
select(Order).where(...).options(selectinload(Order.items))
    ↓
Core builds parameterised SQL
    ↓
Connection taken from the pool
    ↓
Rows returned; mapped to Order objects; identity map deduplicates
    ↓
Business logic mutates objects — changes tracked, no SQL yet
    ↓
commit() → flush: UPDATE/INSERT statements issued → COMMIT
    ↓
Session closed, connection returned to the pool
    ↓
Response built from a Pydantic model, never from the ORM object
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> SQLAlchemy gives you an ORM without giving up SQL — scope the Session to one request, and make loading strategies explicit so N+1 cannot hide.

---

## 20. Common Mistakes

- **A global or long-lived Session** — leaked connections and open transactions
- **Lazy loading in loops** — N+1
- **Not using `raiseload` or `lazy="raise"`** in development
- **Bulk operations through the ORM** instead of Core
- **Mixing sync and async sessions** in one codebase
- **Following 1.x tutorials** while running 2.0
- **Returning ORM models from API endpoints**

---

## 21. Open Source Technologies

- **SQLAlchemy** — Core and ORM
- **Alembic** — migrations
- **asyncpg**, **psycopg 3** — drivers
- **SQLModel** — SQLAlchemy plus Pydantic
- **sqlalchemy-utils**, **pytest fixtures** for session-per-test

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Set `lazy="raise"` on one relationship and run your test suite. What breaks?
- [ ] Enable `echo=True` on the engine and read the SQL one of your endpoints generates.
- [ ] Verify your Session is created and closed per request, not shared.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
App → Session → ORM/Core → SQL → pool → PostgreSQL
```

## 2. Request Flow

```text
Input       a select() statement or object mutation
    ↓
Processing  built into parameterised SQL, executed, mapped, tracked
    ↓
Output      domain objects and, on commit, persisted changes
```

## 3. Real-World Usage

**FastAPI plus SQLAlchemy 2.0 plus Alembic** is the current standard Python API stack. The combination gives type-annotated models, explicit async queries and version-controlled migrations without a full framework.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A Python ORM built on a full SQL expression toolkit |
| **Why does it exist?** | So convenience does not cost control over the query |
| **Where does it belong?** | Between your service layer and the database driver |
| **When should I use it?** | Python applications outside Django, especially with complex queries |

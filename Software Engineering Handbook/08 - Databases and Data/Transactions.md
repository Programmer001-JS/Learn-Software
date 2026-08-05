# Transactions

> **In one line —** a group of operations that all happen or none do, so a crash halfway through cannot leave money in neither account.

| | |
|---|---|
| **Category** | Database Concept |
| **Architectural Layer** | Data |
| **Guarantees** | ACID |
| **Related notes** | [Database Fundamentals](Database%20Fundamentals.md) · [PostgreSQL](PostgreSQL.md) · [Services](../07%20-%20Backend%20Design%20Patterns/Services.md) · [Distributed Lock](Distributed%20Lock.md) · [Microservices](../10%20-%20Distributed%20Systems/Microservices.md) |

---

## 1. Short Definition

*What is it?*

A transaction is a unit of work that the database treats as **indivisible**: every statement in it succeeds and is committed together, or none of them takes effect.

---

## 2. Purpose

*What is its main purpose?*

To keep data correct in the presence of two hostile realities: **crashes** and **concurrency**.

---

## 3. Problem

*What engineering problem does it solve?*

```text
Transfer €100 from A to B

UPDATE accounts SET balance = balance - 100 WHERE id = A;
    ↓  ← crash here
UPDATE accounts SET balance = balance + 100 WHERE id = B;

Without a transaction: €100 has ceased to exist.
With a transaction:    the first statement is rolled back. Nothing happened.
```

---

## 4. ACID

| | Meaning | Failure without it |
|---|---|---|
| **Atomicity** | All or nothing | Half-completed operations |
| **Consistency** | Constraints hold after commit | Invalid data states |
| **Isolation** | Concurrent transactions do not see partial work | Reading half-written data |
| **Durability** | Committed means survived | Confirmed writes lost on power failure |

---

## 5. Architecture Position

```text
Controller             no transaction knowledge
    ↓
SERVICE                BEGIN ... COMMIT      ← the correct boundary
    ↓
Repository             participates
    ↓
Database               enforces isolation and durability
```

> [!IMPORTANT]
> **One use case, one transaction.** The [service layer](../07%20-%20Backend%20Design%20Patterns/Services.md) is the right boundary: a controller does not know which writes belong together, and a repository is too fine-grained to decide.

---

## 6. Isolation levels

The trade is always the same: **stronger isolation, less concurrency**.

```text
READ UNCOMMITTED   sees uncommitted data          — essentially never used
READ COMMITTED     sees only committed data       — PostgreSQL default
REPEATABLE READ    the same query returns the same rows   — MySQL default
SERIALIZABLE       as if transactions ran one at a time   — safest, slowest
```

| Anomaly | Prevented from |
|---|---|
| **Dirty read** — reading uncommitted data | READ COMMITTED |
| **Non-repeatable read** — a row changes mid-transaction | REPEATABLE READ |
| **Phantom read** — new rows appear mid-transaction | SERIALIZABLE |
| **Lost update** — two writers, one overwrites the other | Needs explicit locking |

---

## 7. The lost update — the one that bites in practice

```text
Both transactions read stock = 10
    ↓
Both compute 10 - 1 = 9
    ↓
Both write 9
    ↓
Two items sold, stock decremented once
```

Three correct fixes:

```sql
-- 1. Let the database do the arithmetic (best when possible)
UPDATE stock SET qty = qty - 1 WHERE id = 42 AND qty > 0;

-- 2. Pessimistic lock
SELECT qty FROM stock WHERE id = 42 FOR UPDATE;

-- 3. Optimistic lock (version column)
UPDATE stock SET qty = 9, version = 6 WHERE id = 42 AND version = 5;
-- 0 rows affected → someone else changed it → retry
```

> [!CAUTION]
> Read-modify-write in application code is **not** protected by a transaction alone at READ COMMITTED. This is the most common real-world transaction bug, and it costs real money in inventory and balance systems.

---

## 8. Real World Example

- **Banking and payments** — the textbook case, and still the clearest.
- **Inventory** — overselling is a lost update, not a caching problem.
- **Anything spanning two tables** — creating an order plus its line items must be atomic.
- **[Hibernate](Hibernate.md) and [EF Core](Entity%20Framework.md)** wrap a whole service method in a transaction, which is why an exception rolls back every write in it.

---

## 9. Keep transactions short

```text
BEGIN
    ↓
call an external payment API (2 seconds)     ← ✗ NEVER do this
    ↓
COMMIT
```

> [!CAUTION]
> A long transaction holds locks, blocks other writers, and in PostgreSQL prevents [autovacuum](PostgreSQL.md) from cleaning any row version newer than it — causing table bloat that persists long after the transaction ends.
>
> **Never call an external service inside a transaction.** Do the external work outside, and use an idempotency key so retries are safe.

---

## 10. Distributed transactions

```text
Order service     writes to DB 1
Payment service   writes to DB 2
    ↓
Two-phase commit?  → slow, fragile, blocks on coordinator failure
    ↓
The practical answer: DO NOT distribute the transaction.
Use the SAGA pattern — a sequence of local transactions,
each with a compensating action if a later step fails.
```

> [!IMPORTANT]
> ACID across services is effectively unavailable in practice. This is one of [microservices](../10%20-%20Distributed%20Systems/Microservices.md)' largest hidden costs: an operation that was one transaction in a monolith becomes a saga with compensations, retries and eventual consistency.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Wrap any group of writes that must be consistent together. Use `SELECT ... FOR UPDATE` or atomic SQL wherever a read-modify-write is involved.

> [!CAUTION]
> Do not wrap read-only work in an explicit transaction unnecessarily, do not hold one open across user interaction, and do not include external calls, file uploads or queue publishing inside one.

---

## 12. Advantages and Disadvantages

**Advantages**
- Atomicity across multiple statements
- Isolation from concurrent writers
- Durability guaranteed on commit
- Automatic rollback on error

**Disadvantages**
- Locks reduce concurrency
- Long transactions cause blocking and bloat
- Deadlocks are possible and must be retried
- Do not extend across services or systems
- Isolation levels are subtle and easy to misjudge

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Short transactions** | Minimal cost |
| **Long transactions** | Lock contention, vacuum blocking, connection held |
| **SERIALIZABLE** | Highest correctness, most retries |
| **Deadlocks** | The database kills one participant — your code must retry |
| **Batching writes** | One transaction for 1,000 inserts is far faster than 1,000 transactions |

---

## 14. Security Considerations

> [!CAUTION]
> **Race conditions in transactional logic are exploitable.** Concurrent requests to redeem a single-use coupon, withdraw a balance, or claim a limited resource are a standard attack technique — and they succeed precisely when the code does read-modify-write without a lock or atomic update.

- **Enforce invariants in the database** — unique constraints and `CHECK` constraints cannot be raced
- **Use atomic SQL** (`qty = qty - 1 WHERE qty > 0`) rather than computing in application code
- **Idempotency keys** protect against retried requests creating duplicate charges
- **Audit records should be written in the same transaction** as the action they describe, or they can disagree with reality

---

## 15. Mental Model

> [!NOTE]
> **A transaction is a pencil draft you can erase entirely.**
>
> You write several changes in pencil. If anything goes wrong, you rub out the whole page — not half of it. Only when you commit does the ink go down, and from that point it survives the building burning down.

---

## 16. Mini Architecture Diagram

```text
BEGIN
  ↓
statement 1 ─┐
statement 2 ─┤ all visible only to this transaction
statement 3 ─┘
  ↓
COMMIT  → durable, visible to everyone
   or
ROLLBACK → as if nothing happened
```

---

## 17. Complete Request Flow

Creating an order:

```text
Service method begins a transaction
    ↓
SELECT stock FOR UPDATE          ← locks the row against concurrent writers
    ↓
Stock insufficient → raise → ROLLBACK, nothing written
    ↓
INSERT order
INSERT order_items
UPDATE stock SET qty = qty - n
    ↓
COMMIT — write-ahead log flushed to disk FIRST, then acknowledged
    ↓
Only AFTER commit: publish the OrderCreated event, send the email
    ↓  (inside the transaction, a rollback would have announced a
       non-existent order and sent an email for it)
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> A transaction makes a group of writes all-or-nothing — keep it short, never include external calls, and use atomic SQL or explicit locks for read-modify-write.

---

## 19. Common Mistakes

- **Read-modify-write without a lock** — the lost update
- **External API calls inside a transaction**
- **Long-running transactions** blocking writers and vacuum
- **No deadlock retry** — deadlocks are normal and must be handled
- **Publishing events before commit** — announcing things that may be rolled back
- **Assuming a transaction spans services**
- **One transaction per row** in bulk operations

---

## 20. Open Source Technologies

- **PostgreSQL**, **MySQL/InnoDB** — MVCC transaction engines
- **`SELECT ... FOR UPDATE`**, **`SKIP LOCKED`** — explicit locking
- **Spring `@Transactional`**, SQLAlchemy sessions, EF Core `SaveChanges`
- **Outbox pattern** — publish events atomically with the write
- **Temporal**, **Camunda** — saga orchestration across services

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Find one read-modify-write in your code and check whether concurrent requests could produce a lost update.
- [ ] Search for any external HTTP call made inside a transaction.
- [ ] Verify your application retries on deadlock rather than failing the request.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Service (transaction boundary)
    ↓
Repository writes
    ↓
Database: isolation + WAL + commit
```

## 2. Request Flow

```text
Input       a group of writes that belong together
    ↓
Processing  executed under isolation; locks held; log flushed before commit
    ↓
Output      all changes durable, or none of them
```

## 3. Real-World Usage

**Overselling in e-commerce** is the everyday form of the lost update. Two customers buy the last item simultaneously, both requests read the same stock value, and the fix is an atomic decrement or a row lock — not a bigger cache.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An indivisible group of database operations |
| **Why does it exist?** | Because crashes and concurrency corrupt naive writes |
| **Where does it belong?** | At the service layer, around one use case |
| **When should I use it?** | Whenever several writes must be consistent together |

# Database Fundamentals

> **In one line —** a database is a program whose entire design is an answer to one fact: the disk is slow and the data must survive a crash.

| | |
|---|---|
| **Category** | Overview note *(hub for this section)* |
| **Architectural Layer** | Data |
| **Related notes** | [SQL](SQL.md) · [PostgreSQL](PostgreSQL.md) · [MongoDB](MongoDB.md) · [Transactions](Transactions.md) · [Indexes](Indexes.md) · [SSD and HDD](../02%20-%20Computer%20Science%20Fundamentals/SSD%20and%20HDD.md) |

---

## 1. Short Definition

A database is a system for storing data durably, querying it efficiently, and keeping it correct when many users change it at the same time. Those three requirements — durability, efficient access, concurrency — explain nearly every feature it has.

---

## 2. The two families

```text
RELATIONAL  (SQL)                     NON-RELATIONAL  (NoSQL)
tables, rows, columns                 documents, key-value, graph, columnar
fixed schema, enforced                flexible or no schema
joins across tables                   denormalised, embedded data
ACID transactions                     often eventual consistency
PostgreSQL, MySQL                     MongoDB, Redis, Cassandra, Neo4j
```

> [!IMPORTANT]
> **Start with PostgreSQL unless you have a specific reason not to.** It handles relational data, JSON documents, full-text search, geospatial queries and vectors. Most "we need NoSQL" decisions are made before the actual access patterns are known.

---

## 3. ACID — what a relational database guarantees

| | Meaning |
|---|---|
| **Atomicity** | All of a transaction happens, or none of it |
| **Consistency** | Constraints are never violated by a committed transaction |
| **Isolation** | Concurrent transactions do not see each other's partial work |
| **Durability** | Once committed, it survives a crash |

```text
Transfer €100 from A to B
    ↓
debit A  ─┐
          ├─ both, or neither. A crash between them must not lose €100.
credit B ─┘
```

See [Transactions](Transactions.md).

---

## 4. Architecture Position

```text
Application
    ↓
Connection pool          ← connections are expensive; never one per request
    ↓
Cache (Redis)            ← answers most reads before they reach the database
    ↓
DATABASE
    ├── query parser and planner
    ├── buffer pool (RAM)     ← avoids disk reads
    ├── indexes               ← avoids full scans
    └── write-ahead log       ← makes crashes survivable
    ↓
Disk
```

---

## 5. Why databases are shaped the way they are

Everything follows from the [memory hierarchy](../02%20-%20Computer%20Science%20Fundamentals/01%20-%20How%20Computers%20Work.md):

```text
RAM read    ~100 ns
SSD read    ~100 µs      ← 1,000× slower
```

- **Indexes** exist so a query touches a few pages instead of scanning a table
- **The buffer pool** keeps hot pages in RAM so most reads never reach the disk
- **The write-ahead log** turns random writes into sequential ones, which are far cheaper
- **Query planners** exist because the same result can be computed in ways that differ by orders of magnitude

---

## 6. Normalisation and its limits

```text
NORMALISED                            DENORMALISED
each fact stored once                 facts duplicated for read speed
updates are simple and safe           updates must touch many places
reads need joins                      reads are single-row
    ↓                                     ↓
correctness first                     read performance first
```

> [!TIP]
> Normalise by default; denormalise deliberately, with a written reason, when a measured read pattern demands it. Denormalising early gives you the update anomalies without the traffic that would have justified them.

---

## 7. Where things go wrong

| Problem | Cause |
|---|---|
| **N+1 queries** | One query per row in a loop — the most common performance bug |
| **Missing index** | A full table scan on every request |
| **Connection exhaustion** | Pool size × workers × replicas exceeds `max_connections` |
| **Lock contention** | Long transactions holding rows other requests need |
| **Unbounded result sets** | `SELECT *` with no `LIMIT` on a growing table |

---

## 8. Real World Example

- **Instagram** ran on PostgreSQL far longer than expected, sharding only when genuinely forced to.
- **Stack Overflow** served enormous traffic from one SQL Server, because caching removed most reads.
- **Every "we need NoSQL for scale" story** should be checked against the fact that a single well-indexed PostgreSQL instance comfortably handles tens of thousands of queries per second.

---

## 9. Mental Model

> [!NOTE]
> **A database is a library with a card catalogue and a strict lending record.**
>
> The catalogue ([index](Indexes.md)) means you do not walk every shelf. The lending record ([transactions](Transactions.md)) means two people cannot borrow the same book. The reading room ([buffer pool](../02%20-%20Computer%20Science%20Fundamentals/RAM.md)) holds what is being used right now, because fetching from the stacks is slow.

---

## 10. Key Takeaway

> [!IMPORTANT]
> A database exists to make disk-backed data fast to query and safe to change concurrently — and almost every performance problem is a missing index, an N+1 query, or a transaction held too long.

---

## 11. Common Mistakes

- **Choosing NoSQL before knowing the access patterns**
- **No connection pooling**, or a pool sized without counting workers and replicas
- **N+1 queries** hidden behind an ORM
- **No indexes on foreign keys and filter columns**
- **Long-running transactions** holding locks
- **Storing files in the database** instead of object storage
- **No backup restore test** — an untested backup is a hope, not a plan

---

## 12. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 13. Workbook Exercise

- [ ] Find your slowest query and run `EXPLAIN ANALYZE` on it.
- [ ] Count total possible connections: pool size × workers × replicas, against your database limit.
- [ ] Actually restore one backup into a scratch database and time how long it takes.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Application → connection pool → cache → database (planner, buffer pool, indexes, WAL) → disk
```

## 2. Request Flow

```text
Input       a query
    ↓
Processing  parsed, planned, served from the buffer pool or read from disk via an index
    ↓
Output      rows, with changes durably logged before they are acknowledged
```

## 3. Real-World Usage

**Instagram** delayed sharding PostgreSQL for years by indexing carefully and caching aggressively. Most teams reach for a distributed database long before their single instance is actually the limit.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A system for durable, queryable, concurrently modified data |
| **Why does it exist?** | Because disks are slow and concurrent writes corrupt naive storage |
| **Where does it belong?** | Behind a cache, behind a connection pool, beneath your application |
| **When should I use it?** | For anything that must persist — starting with PostgreSQL by default |

# Indexes

> **In one line —** a sorted lookup structure that turns "read every row" into "read three pages" — and the single highest-leverage database optimisation there is.

| | |
|---|---|
| **Category** | Database Structure |
| **Architectural Layer** | Data |
| **Related notes** | [SQL](SQL.md) · [PostgreSQL](PostgreSQL.md) · [Database Optimization](Database%20Optimization.md) · [Database Fundamentals](Database%20Fundamentals.md) |

---

## 1. Short Definition

*What is it?*

An index is a separate, sorted data structure that maps column values to row locations, so the database can find matching rows without scanning the whole table.

---

## 2. Purpose

*What is its main purpose?*

To reduce the number of disk pages a query must read. Everything else about indexing follows from that one goal.

---

## 3. Problem

*What engineering problem does it solve?*

```text
SELECT * FROM users WHERE email = 'ana@example.com';

WITHOUT AN INDEX                      WITH AN INDEX
sequential scan                       B-tree lookup
10,000,000 rows read                  ~3 pages read
~4 seconds                            ~0.2 milliseconds
```

The ratio grows with the table. An unindexed query that is fine at 10,000 rows is an outage at 10 million.

---

## 4. How a B-tree index works

```text
                    [ M ]
                   /     \
            [ D  H ]     [ R  W ]
            /  |  \       /  |  \
        leaves: sorted values → row pointers
```

```text
Each level narrows the search
    ↓
~3–4 levels covers millions of rows
    ↓
3–4 page reads instead of a full scan
```

This is also why a B-tree serves **ranges** and **sorting** as well as equality — the leaves are in order.

---

## 5. Index types

| Type | Good for | Notes |
|---|---|---|
| **B-tree** | Equality, ranges, `ORDER BY` | The default; covers most needs |
| **Hash** | Exact equality only | Rarely worth choosing over B-tree |
| **GIN** | JSONB, arrays, full-text | "Does this contain that?" |
| **GiST** | Geospatial, ranges, nearest-neighbour | PostGIS depends on it |
| **BRIN** | Enormous, naturally ordered tables | Tiny index, e.g. time-series |
| **HNSW / IVFFlat** | Vector similarity | Via [pgvector](../11%20-%20AI%20Engineering/pgvector.md) |

---

## 6. Composite indexes and column order

```sql
CREATE INDEX idx ON orders (user_id, status, created);
```

```text
✓ WHERE user_id = 1
✓ WHERE user_id = 1 AND status = 'pending'
✓ WHERE user_id = 1 AND status = 'pending' ORDER BY created
✗ WHERE status = 'pending'                 ← cannot use it: user_id is missing
```

> [!IMPORTANT]
> **A composite index works left-to-right, like a phone book sorted by surname then first name.** You can look up "Smith", and "Smith, John" — but not "everyone called John". Column order is the single most important decision when creating one.

**Rule of thumb:** equality columns first, then range columns, then sort columns.

---

## 7. Partial and expression indexes — the underused ones

```sql
-- Index only the rows you actually query
CREATE INDEX idx_pending ON orders (created)
  WHERE status = 'pending';          -- maybe 2% of the table

-- Index the shape you query, not the raw column
CREATE INDEX idx_email_lower ON users (lower(email));
```

> [!CAUTION]
> **A function on an indexed column disables the index.** `WHERE lower(email) = 'x'` cannot use a plain index on `email`; it needs the expression index above. The same applies to `WHERE date(created) = ...` and to implicit type casts.

---

## 8. Covering indexes

```sql
CREATE INDEX idx ON orders (user_id) INCLUDE (total, status);
```

If every column a query needs is in the index, the database never touches the table at all — an **index-only scan**. This is also the strongest argument against `SELECT *`: it guarantees the table must be read.

---

## 9. What indexes cost

> [!IMPORTANT]
> Indexes are not free. Every one of them must be updated on every `INSERT`, `UPDATE` and `DELETE`.

```text
A table with 10 indexes
    ↓
One INSERT = 1 table write + 10 index writes
    ↓
Write throughput drops substantially
```

```text
✓ Index:  foreign keys · WHERE columns · JOIN columns · ORDER BY columns
✗ Do not index:  low-cardinality columns (a boolean over 50/50 data)
                 columns you never filter on
                 small tables (a scan is already cheap)
                 every column "just in case"
```

---

## 10. Real World Example

- **PostgreSQL does not automatically index foreign keys.** This is the most common missing index in real systems, and it makes joins and cascading deletes extremely slow.
- **`EXPLAIN ANALYZE`** shows immediately whether an index was used; "Seq Scan" on a large table is the signal.
- **`pg_stat_user_indexes`** reveals indexes that have never been used — pure write overhead.

---

## 11. Communication and Dependencies

- **The query planner** decides whether to use an index, based on **statistics**
- **`ANALYZE`** keeps those statistics current; stale statistics cause bad plans
- **Disk and memory** — indexes occupy real space and compete for the buffer pool

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Index every foreign key, and every column that appears in a `WHERE`, `JOIN` or `ORDER BY` on a table of meaningful size. Then measure.

> [!CAUTION]
> Do not add indexes speculatively. Each one slows writes, consumes memory and space, and gives the planner another option to choose wrongly. Add them in response to a measured query, not to a hypothesis.

---

## 13. Advantages and Disadvantages

**Advantages**
- Orders of magnitude faster reads
- Enable efficient sorting and range queries
- Unique indexes enforce correctness, not just speed
- Index-only scans avoid touching the table entirely

**Disadvantages**
- Every write must update every index
- Disk and memory cost
- Unused indexes are pure overhead
- Creating one on a large table can lock it — use `CREATE INDEX CONCURRENTLY`
- Too many options can lead the planner astray

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Reads** | 10–10,000× faster on large tables |
| **Writes** | Each index adds real cost |
| **Storage** | Frequently 10–30% of the table size |
| **Memory** | Hot index pages compete for the buffer pool |
| **Creation** | Locks the table unless created concurrently |

---

## 15. Security Considerations

> [!CAUTION]
> **A missing index is a denial-of-service vector.** An endpoint that triggers a sequential scan over ten million rows can be called repeatedly by anyone, saturating the database and taking the whole application down. Unindexed search and filter endpoints are a genuine availability risk, not merely a performance annoyance.

- **Unique indexes enforce security invariants** — one account per email, one redemption per coupon — in a way application code cannot be raced past
- **Timing differences are observable**: an indexed lookup that returns instantly for existing accounts and slowly for missing ones leaks which accounts exist
- **Rate-limit expensive query endpoints** regardless of indexing

---

## 16. Mental Model

> [!NOTE]
> **An index is the index at the back of a book.**
>
> Without it, finding a topic means reading every page. With it, you look up the word and jump straight to page 247. And the reason a book has one index rather than twelve is the same reason a table should not: every index must be rebuilt whenever the book changes.

---

## 17. Mini Architecture Diagram

```text
Query
    ↓
Planner consults statistics
    ↓
Index available and selective?
    ├─ YES → index scan  → a few pages
    └─ NO  → sequential scan → every page
    ↓
Buffer pool → disk
```

---

## 18. Complete Request Flow

```text
SELECT * FROM orders
WHERE user_id = 42 AND status = 'pending'
ORDER BY created DESC LIMIT 20;
    ↓
Planner: is there an index on (user_id, status, created)?
    ↓ YES
Index scan: descend the B-tree to user_id=42, status='pending'
    ↓
Leaves are already ordered by created → NO SORT NEEDED
    ↓
Read 20 entries, fetch those rows from the table
    ↓
~0.3 ms
    ↓
─────────── without the index ───────────
Sequential scan: read all 10,000,000 rows
    ↓
Filter, then sort the survivors
    ↓
~4 seconds, and the buffer pool is now full of useless pages
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> An index turns a full scan into a handful of page reads, and costs you on every write — index what you actually filter, join and sort on, and verify with `EXPLAIN ANALYZE`.

---

## 20. Common Mistakes

- **No index on foreign keys** — the most common missing index in production
- **Wrong column order** in a composite index
- **Functions on indexed columns** (`lower(email)`, `date(created)`) disabling the index
- **Indexing everything**, crippling write throughput
- **Never removing unused indexes**
- **Creating an index on a large table without `CONCURRENTLY`**, locking it
- **Never reading the query plan** before or after adding one

---

## 21. Open Source Technologies

- **`EXPLAIN ANALYZE`** — the primary tool
- **pg_stat_user_indexes**, **pg_stat_statements** — usage and slow queries
- **HypoPG** — test a hypothetical index before creating it
- **pgvector**, **PostGIS** — specialised index types
- **pt-index-usage** (Percona) for MySQL

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Check whether every foreign key in your schema has an index.
- [ ] Run `EXPLAIN ANALYZE` on your slowest query and identify Seq Scan versus Index Scan.
- [ ] Query `pg_stat_user_indexes` for indexes with zero scans and consider dropping them.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Query → planner → index scan or sequential scan → buffer pool → disk
```

## 2. Request Flow

```text
Input       a query with filter, join or sort conditions
    ↓
Processing  planner chooses an index if one is selective and usable
    ↓
Output      rows, read from a few pages instead of the whole table
```

## 3. Real-World Usage

**Missing foreign key indexes** are the single most common performance defect in production PostgreSQL databases, because — unlike MySQL — PostgreSQL does not create them automatically and nothing warns you.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A sorted structure mapping column values to row locations |
| **Why does it exist?** | Because reading every row does not scale |
| **Where does it belong?** | On columns used in filters, joins and sorts |
| **When should I use it?** | In response to a measured query — never speculatively |

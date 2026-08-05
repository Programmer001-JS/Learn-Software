# Database Optimization

> **In one line —** measure first, then fix the query — because the answer is almost always an index or an N+1, and almost never a bigger server.

| | |
|---|---|
| **Category** | Practice |
| **Architectural Layer** | Data |
| **Related notes** | [Indexes](Indexes.md) · [SQL](SQL.md) · [ORM](ORM.md) · [PostgreSQL](PostgreSQL.md) · [Cache](Cache.md) · [Performance Engineering](../14%20-%20Scalability%20and%20Reliability/Performance%20Engineering.md) |

---

## 1. Short Definition

*What is it?*

Database optimisation is the practice of finding which queries actually consume time and fixing them — in a defined order, based on measurements rather than intuition.

---

## 2. Purpose

*What is its main purpose?*

To spend effort where it changes something. Most database work is guesswork; the discipline is in refusing to change anything before you know what is slow.

---

## 3. The order of operations

> [!IMPORTANT]
> Work down this list. Each step is roughly ten times more effort than the one above it, and roughly ten times less likely to be necessary.

```text
1. MEASURE            find the actual slow queries
    ↓
2. FIX N+1            usually the largest single win in an ORM application
    ↓
3. ADD INDEXES        the second largest, and cheap
    ↓
4. REWRITE QUERIES    select fewer columns, avoid functions on indexed columns
    ↓
5. CACHE              only after the query itself is reasonable
    ↓
6. TUNE CONFIGURATION shared_buffers, work_mem, pool size
    ↓
7. READ REPLICAS      scale reads horizontally
    ↓
8. DENORMALISE        deliberately trade correctness convenience for speed
    ↓
9. SHARD              the last resort; enormous operational cost
```

---

## 4. Step 1 — Measure

```sql
-- PostgreSQL: which queries consume the most total time?
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

> [!TIP]
> Order by **total** time, not mean. A query taking 5 ms called 200,000 times costs far more than one taking 2 seconds called twice — and the fast-looking one is usually the real problem.

Then `EXPLAIN ANALYZE` the top offenders. `Seq Scan` on a large table is your answer.

---

## 5. Step 2 — N+1 queries

The most common cause of a slow page in any ORM application:

```text
100 orders, each loading its customer separately
    ↓
101 queries × 2 ms round trip = ~200 ms of pure waiting
    ↓
Eager loading → 1 query → ~5 ms
```

See [ORM](ORM.md) for the fix in each framework. A query counter in development makes this visible immediately.

---

## 6. Step 3 — Indexes

Covered fully in [Indexes](Indexes.md). The short version:

```text
Index every foreign key
Index every column in WHERE, JOIN and ORDER BY
Composite index column order: equality → range → sort
Functions on columns disable indexes
Remove indexes with zero scans
```

---

## 7. Step 4 — Query rewriting

```sql
-- ✗ fetches every column; blocks index-only scans
SELECT * FROM orders WHERE user_id = 42;

-- ✓ only what is needed
SELECT id, total FROM orders WHERE user_id = 42;

-- ✗ function disables the index
WHERE date(created) = '2026-01-01'
-- ✓ range condition uses it
WHERE created >= '2026-01-01' AND created < '2026-01-02'

-- ✗ offset pagination degrades: OFFSET 100000 reads 100,020 rows
LIMIT 20 OFFSET 100000
-- ✓ keyset pagination stays constant
WHERE id < :last_seen ORDER BY id DESC LIMIT 20
```

> [!TIP]
> **Keyset pagination** is one of the highest-value rewrites available. Offset pagination gets linearly slower as users page deeper; keyset pagination does not degrade at all.

---

## 8. Step 5 — Caching

Only once the query itself is sensible. See [Cache](Cache.md).

> [!CAUTION]
> **Caching a slow query hides a missing index rather than fixing it.** The cold cache, the cache restart and the next scale-up will all reveal it — usually at the worst moment.

---

## 9. Step 6 — Configuration

| Setting | Guidance |
|---|---|
| **shared_buffers** | ~25% of RAM (PostgreSQL) |
| **innodb_buffer_pool_size** | ~70–80% of RAM (MySQL) |
| **work_mem** | Per sort/hash operation — too high multiplies across concurrent queries |
| **max_connections** | Low, with a pooler in front |
| **effective_cache_size** | Tells the planner how much OS cache to assume |

> [!TIP]
> Default configurations are conservative and assume a small machine. On a server with 64 GB of RAM, defaults leave most of it unused.

---

## 10. Step 7–9 — Scaling

```text
READ REPLICAS      scale reads; beware replica lag on read-after-write
    ↓
DENORMALISATION    duplicate data to avoid joins; you now own consistency
    ↓
PARTITIONING       split one huge table by range (usually time)
    ↓
SHARDING           split across machines — the point of no return
```

> [!CAUTION]
> **Sharding is the last resort.** It breaks joins, complicates transactions, makes every query need a shard key, and is extremely hard to undo. Exhaust everything above it first — a well-indexed single PostgreSQL instance handles far more than most teams assume.

---

## 11. Real World Example

- **Instagram** deferred sharding for years by indexing and caching properly.
- **The typical "the database is slow" incident** resolves to one missing index, found in five minutes with `EXPLAIN ANALYZE`.
- **Connection exhaustion** is frequently misdiagnosed as slowness — the queries are fast, the requests are queuing for a connection.

---

## 12. Connection pool sizing

```text
Total connections = pool_size × workers × replicas
    ↓
16 replicas × 4 workers × 10 = 640
    ↓
PostgreSQL max_connections = 100
    ↓
Outage under load
```

> [!TIP]
> More connections is not more throughput. A pool larger than the database can usefully serve just moves the queue from your application into the database. Use PgBouncer and keep pools small.

---

## 13. When To Use / When NOT To Use

> [!TIP]
> Optimise when you have a measurement showing a specific query is a problem. Fix that query, measure again, and stop when it is fast enough.

> [!CAUTION]
> Do not optimise speculatively, do not denormalise before measuring, and do not add infrastructure to avoid fixing a query. Premature denormalisation gives you update anomalies without the traffic that would have justified them.

---

## 14. Security Considerations

> [!CAUTION]
> **An unoptimised query is an availability vulnerability.** An endpoint that triggers a full scan over ten million rows can be invoked repeatedly by anyone, exhausting the database and taking down the entire application. Search, filter and export endpoints are the usual candidates.

- **Set `statement_timeout`** so a runaway query cannot run indefinitely
- **Cap page sizes** — an unbounded `limit` parameter is a resource-exhaustion vector
- **Rate-limit expensive endpoints**, independently of indexing
- **Never expose query errors** to clients; they reveal schema details

---

## 15. Mental Model

> [!NOTE]
> **Database optimisation is diagnosing a patient, not prescribing vitamins.**
>
> You take measurements first. The overwhelming majority of cases have one specific, identifiable cause. Buying a bigger server before diagnosis is treating a symptom you have not located, at considerable cost.

---

## 16. Mini Architecture Diagram

```text
Slow page reported
    ↓
pg_stat_statements → the actual worst queries
    ↓
EXPLAIN ANALYZE → Seq Scan? sort? N+1?
    ↓
Fix: eager load · add an index · rewrite
    ↓
Measure again
    ↓
Still slow? → cache → tune → replicas → shard
```

---

## 17. Complete Request Flow

A realistic optimisation session:

```text
"The orders page takes 4 seconds"
    ↓
Query log: 340 queries on that page          ← N+1, found in one minute
    ↓
Add eager loading → 3 queries → 900 ms
    ↓
EXPLAIN ANALYZE the remaining slow one → Seq Scan on 8M rows
    ↓
CREATE INDEX CONCURRENTLY on (user_id, status, created) → 120 ms
    ↓
Replace SELECT * with the six columns actually used → 90 ms
    ↓
Fast enough. STOP.
    ↓
No cache added, no configuration changed, no replica, no sharding.
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Measure, fix the N+1, add the index — in that order. Almost every database performance problem is resolved before you reach step four.

---

## 19. Common Mistakes

- **Optimising without measuring**
- **Caching to hide a missing index**
- **Adding hardware before diagnosis**
- **Ordering slow queries by mean time** instead of total time
- **Offset pagination** on deep pages
- **Oversized connection pools**
- **Denormalising or sharding early**
- **Not setting `statement_timeout`**

---

## 20. Open Source Technologies

- **pg_stat_statements**, **EXPLAIN ANALYZE**, **auto_explain** — PostgreSQL
- **pgBadger**, **pganalyze**, **PMM** — analysis dashboards
- **django-debug-toolbar**, **Laravel Debugbar**, **MiniProfiler** — query counts in development
- **PgBouncer**, **ProxySQL** — connection pooling
- **HypoPG** — test an index before creating it
- **pgbench**, **sysbench** — load testing

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Enable `pg_stat_statements` and list your top five queries by total execution time.
- [ ] Count the queries on your heaviest page and find any N+1.
- [ ] Check whether `statement_timeout` is set, and what your largest allowed page size is.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Measure → fix N+1 → index → rewrite → cache → tune → replicas → shard
```

## 2. Request Flow

```text
Input       a report that something is slow
    ↓
Processing  measure, locate the query, read the plan, fix the specific cause
    ↓
Output      a measurably faster query — and no unnecessary infrastructure
```

## 3. Real-World Usage

The most common production optimisation is the least glamorous: **one missing index on a foreign key**, found with `EXPLAIN ANALYZE` in minutes, after a team spent a week discussing caching strategies.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A measurement-driven practice for making database access fast |
| **Why does it exist?** | Because intuition about database performance is reliably wrong |
| **Where does it belong?** | Between a performance complaint and any infrastructure change |
| **When should I use it?** | When you have a measurement — never before |

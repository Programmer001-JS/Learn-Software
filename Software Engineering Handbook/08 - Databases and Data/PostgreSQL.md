# PostgreSQL

> **In one line —** the default answer to "which database?" — a relational engine that also does JSON, full-text search, geospatial and vectors well enough that most projects never need a second one.

| | |
|---|---|
| **Category** | Relational Database |
| **Architectural Layer** | Data |
| **Licence** | Open source, permissive |
| **Related notes** | [SQL](SQL.md) · [Database Fundamentals](Database%20Fundamentals.md) · [Transactions](Transactions.md) · [Indexes](Indexes.md) · [pgvector](../11%20-%20AI%20Engineering/pgvector.md) · [Process](../02%20-%20Computer%20Science%20Fundamentals/Process.md) |

---

## 1. Short Definition

*What is it?*

PostgreSQL is an open-source relational database known for correctness, extensibility and standards compliance. It is fully ACID, deeply extensible, and has absorbed capabilities that used to require separate specialised systems.

---

## 2. Purpose

*What is its main purpose?*

To store relational data safely and query it efficiently — and, increasingly, to be the one datastore a project needs rather than one of five.

---

## 3. Problem

*What engineering problem does it solve?*

A typical stack once required PostgreSQL for relations, MongoDB for documents, Elasticsearch for search, PostGIS for geography and a vector database for embeddings. Each is another system to run, secure, back up and keep consistent.

```text
PostgreSQL today handles:
    relations · JSONB documents · full-text search · geospatial (PostGIS)
    vectors (pgvector) · time series (TimescaleDB) · queues (SKIP LOCKED)
```

> [!TIP]
> "Just use Postgres" is a genuine architectural position, not laziness. Every datastore you avoid is one you do not have to operate, monitor, back up or keep in sync.

---

## 4. Architecture Position

```text
Application
    ↓
Connection pool / PgBouncer      ← essential; see section 8
    ↓
┌──────── POSTGRESQL ────────┐
│  postmaster                │  one PROCESS per connection
│  query planner             │
│  shared buffers (RAM)      │
│  WAL (write-ahead log)     │
│  MVCC + autovacuum         │
└──────────────┬─────────────┘
               ↓
             Disk
               ↓
        Streaming replicas
```

---

## 5. MVCC — the central design choice

PostgreSQL uses **Multi-Version Concurrency Control**: an `UPDATE` does not overwrite a row, it writes a new version and marks the old one dead.

```text
Reader sees the version valid at the moment its transaction started
Writer creates a new version
    ↓
READERS NEVER BLOCK WRITERS. WRITERS NEVER BLOCK READERS.
```

> [!IMPORTANT]
> This is why PostgreSQL handles concurrent reads and writes so well — and also why **autovacuum** exists. Dead row versions accumulate; if vacuum cannot keep up, tables bloat and queries slow down. A long-running transaction blocks vacuum from cleaning anything newer than it, which is why an idle-in-transaction connection is genuinely dangerous.

---

## 6. What you get beyond plain SQL

| Feature | Use |
|---|---|
| **JSONB** | Store and index documents, with real query support |
| **Full-text search** | `tsvector`/`tsquery` — enough for most applications |
| **Arrays and ranges** | Native types, indexable |
| **CTEs and window functions** | Complex analytics in one query |
| **Partial and expression indexes** | Index only the rows or shape you query |
| **`SELECT ... FOR UPDATE SKIP LOCKED`** | A correct job queue in plain SQL |
| **Logical replication** | Stream changes to other systems |
| **Extensions** | PostGIS, pgvector, TimescaleDB, pg_stat_statements |

---

## 7. Index types — a real advantage

```text
B-tree     the default: equality, ranges, sorting
GIN        JSONB, arrays, full-text — "does this contain that?"
GiST       geospatial, ranges, nearest-neighbour
BRIN       enormous naturally-ordered tables, tiny index
HNSW/IVFFlat  vector similarity, via pgvector
```

Partial indexes are underused and often the cheapest win available:

```sql
CREATE INDEX idx_pending ON orders (created)
  WHERE status = 'pending';    -- indexes 2% of the table
```

---

## 8. Connections are processes

> [!CAUTION]
> PostgreSQL forks a **separate operating-system process per connection**, each costing several megabytes. The default `max_connections` is 100, and exceeding it is a common production outage.

```text
16 replicas × 4 workers × pool of 10 = 640 connections
    ↓
max_connections = 100
    ↓
"FATAL: sorry, too many clients already"
```

The fix is **PgBouncer** in transaction pooling mode: thousands of client connections multiplexed onto a few dozen real ones.

---

## 9. Real World Example

- **Instagram, Reddit, Apple, Spotify** run PostgreSQL at very large scale.
- **Supabase and Neon** built entire platforms on it, exposing its extensibility as a product.
- **pgvector** made PostgreSQL a credible vector database, removing the need for a separate one in many RAG systems.
- **`SKIP LOCKED`** lets it act as a job queue without adding RabbitMQ.

---

## 10. PostgreSQL vs MySQL

| | PostgreSQL | [MySQL](MySQL.md) |
|---|---|---|
| **Priority** | Correctness and features | Simplicity and read speed |
| **JSON** | JSONB, indexable, rich operators | Weaker |
| **Extensions** | Extensive | Limited |
| **Complex queries** | Better planner, CTEs, windows | Improving |
| **Replication** | Streaming and logical | Very mature |
| **Ubiquity** | High | Higher in shared hosting |

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use PostgreSQL as the default for essentially any new application. The burden of proof should be on choosing something else.

> [!CAUTION]
> - **Extreme write throughput across many nodes** — Cassandra or ScyllaDB are built for that shape.
> - **Pure caching** — [Redis](Redis.md) is the right tool.
> - **Very large-scale analytics** — a columnar warehouse (ClickHouse, BigQuery) will beat it.
> - **Massive numbers of short-lived connections** — workable, but you must plan for PgBouncer.

---

## 12. Advantages and Disadvantages

**Advantages**
- Excellent correctness and standards compliance
- MVCC gives strong concurrent read/write behaviour
- Extensibility covers search, geo, vectors and time series
- Sophisticated query planner
- Free, permissively licensed, no vendor
- Outstanding documentation

**Disadvantages**
- Process-per-connection requires a pooler at scale
- Autovacuum needs understanding and occasional tuning
- Default configuration is conservative for modern hardware
- Major-version upgrades require planning
- Horizontal write scaling needs external tooling (Citus, sharding)

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **shared_buffers** | Typically 25% of RAM; the buffer pool that avoids disk reads |
| **Indexes** | The dominant factor in query time |
| **Connections** | Each is a process — pool aggressively |
| **Autovacuum** | Falling behind causes bloat and gradual slowdown |
| **Replicas** | Read scaling is straightforward; write scaling is not |

---

## 14. Security Considerations

> [!CAUTION]
> **Never let the application connect as a superuser.** A SQL injection with superuser rights is total compromise; with a least-privileged role it is bounded by that role's grants.

- **Row-Level Security** enforces multi-tenancy in the database itself — the strongest defence against a forgotten `WHERE tenant_id = ...`
- **`pg_hba.conf`** controls who may connect from where; require `scram-sha-256`, never `trust`
- **TLS for connections**, especially across networks
- **Encryption at rest** at the disk or cloud-volume level
- **Never expose port 5432 to the internet** — a persistently scanned port
- **Test your restores**; an untested backup is not a backup

---

## 15. Mental Model

> [!NOTE]
> **PostgreSQL is a Swiss Army knife that is also a genuinely good knife.**
>
> Most multi-tools are mediocre at everything. This one is excellent at its main job and surprisingly capable at the others — which is why carrying five separate tools is usually the wrong trade.

---

## 16. Mini Architecture Diagram

```text
App instances
    ↓
PgBouncer  (transaction pooling)
    ↓
PostgreSQL primary
  ├─ planner → executor
  ├─ shared buffers (RAM)
  ├─ WAL → disk
  └─ autovacuum
    ↓ streaming replication
Read replicas
```

---

## 17. Complete Request Flow

```text
Query arrives over a pooled connection
    ↓
Parsed and planned; the planner consults statistics
    ↓
Index scan chosen (or a sequential scan if no index fits)
    ↓
Pages read from shared buffers, or from disk on a miss
    ↓
MVCC: only row versions visible to this transaction are returned
    ↓
On write: WAL entry flushed FIRST, then the change applied
    ↓
COMMIT acknowledged only after the WAL is durable
    ↓
Old row versions left behind → autovacuum reclaims them later
    ↓
WAL streamed to replicas
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> PostgreSQL is correct, extensible and capable enough to be the only datastore most projects need — and its two operational essentials are connection pooling and a healthy autovacuum.

---

## 19. Common Mistakes

- **No connection pooler**, then hitting `max_connections`
- **Idle-in-transaction connections** blocking vacuum and bloating tables
- **Running with default configuration** on a large machine
- **Missing indexes** on foreign keys — Postgres does not create them automatically
- **Connecting as superuser** from the application
- **Ignoring autovacuum** until queries slow down mysteriously
- **Never testing a restore**

---

## 20. Open Source Technologies

- **PostgreSQL** — the database
- **PgBouncer**, **pgpool** — connection pooling
- **pgvector**, **PostGIS**, **TimescaleDB**, **Citus** — extensions
- **pg_stat_statements**, **pgBadger**, **pganalyze** — query analysis
- **pgBackRest**, **WAL-G** — backup and point-in-time recovery
- **Supabase**, **Neon** — managed platforms built on it

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Run `pg_stat_statements` and identify your five slowest queries by total time.
- [ ] Check for any connection sitting in `idle in transaction` state.
- [ ] Verify that every foreign key column in your schema has an index.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
App → PgBouncer → PostgreSQL (planner, buffers, WAL, MVCC) → disk → replicas
```

## 2. Request Flow

```text
Input       a SQL query on a pooled connection
    ↓
Processing  planned, executed against buffers and indexes, MVCC-filtered
    ↓
Output      rows; writes durable in the WAL before commit is acknowledged
```

## 3. Real-World Usage

**pgvector** turned PostgreSQL into a viable vector store, so many RAG systems now run embeddings alongside their relational data in one database. That pattern — absorbing a specialised workload rather than adding a system — is PostgreSQL's recurring story.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An extensible, ACID-compliant open-source relational database |
| **Why does it exist?** | To store relational data correctly, and increasingly everything else too |
| **Where does it belong?** | Behind a connection pooler, beneath your application |
| **When should I use it?** | As the default — choose something else only with a specific reason |

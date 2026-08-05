# MySQL

> **In one line —** the most widely deployed relational database in the world, built for simplicity and read speed, and the default on essentially every shared host.

| | |
|---|---|
| **Category** | Relational Database |
| **Architectural Layer** | Data |
| **Owner** | Oracle (with MariaDB as the community fork) |
| **Related notes** | [PostgreSQL](PostgreSQL.md) · [SQL](SQL.md) · [Transactions](Transactions.md) · [Indexes](Indexes.md) · [Laravel](../06%20-%20Backend%20Architecture/Laravel.md) |

---

## 1. Short Definition

*What is it?*

MySQL is an open-source relational database. Its default storage engine, **InnoDB**, provides ACID transactions, row-level locking and MVCC — everything a typical application needs.

---

## 2. Purpose

*What is its main purpose?*

To be a fast, simple, universally available relational database. It powered the original LAMP stack and remains the default for WordPress, most PHP applications and a large share of shared hosting.

---

## 3. Problem

*What engineering problem does it solve?*

In the late 1990s, relational databases were expensive commercial products. MySQL was free, easy to install, and fast enough for the read-heavy web applications of the time — which is how it became ubiquitous.

---

## 4. Architecture Position

```text
Application
    ↓
Connection pool
    ↓
┌──────── MySQL ────────┐
│  connection THREADS   │  ← threads, not processes
│  parser + optimiser   │
│  storage engine layer │  ← pluggable; use InnoDB
│    InnoDB:            │
│      buffer pool      │
│      redo log         │
│      MVCC             │
└──────────┬────────────┘
           ↓
         Disk
           ↓
      Replicas
```

> [!IMPORTANT]
> The **pluggable storage engine** is MySQL's distinctive design. In practice there is one correct choice: **InnoDB**. MyISAM has no transactions and no crash safety — if you find a table using it, that is a bug, not a decision.

---

## 5. MySQL vs PostgreSQL

| | MySQL | [PostgreSQL](PostgreSQL.md) |
|---|---|---|
| **Connections** | Threads — lighter | Processes — needs pooling sooner |
| **Simple reads** | Very fast | Fast |
| **Complex queries** | Weaker planner | Stronger planner, CTEs, windows |
| **JSON** | Functional, less capable | JSONB, indexable, rich |
| **Extensions** | Few | Extensive |
| **Replication** | Extremely mature, widely operated | Streaming and logical |
| **Ubiquity** | Higher — shared hosting default | High and growing |

> [!TIP]
> For a **new** project, PostgreSQL is usually the better default. Choose MySQL when the ecosystem demands it (WordPress, an existing PHP stack), when the team knows its operations deeply, or when a managed offering you are already using is MySQL-based.

---

## 6. The gotchas that surprise people

> [!CAUTION]
> MySQL historically prioritised "keep going" over "be correct", and some of that survives in defaults.

```text
utf8         → only 3 bytes; cannot store emoji. Use utf8mb4 ALWAYS.
STRICT mode  → off in older versions: '2024-13-45' silently became '0000-00-00'
Table names  → case sensitivity depends on the operating system
DDL          → historically not transactional; a failed migration could leave
               the schema half-changed
```

Modern MySQL 8 fixed most of this — strict mode by default, `utf8mb4` default, atomic DDL. But **an inherited database may not be on modern defaults**, and checking is worth five minutes.

---

## 7. Replication

MySQL's replication is its strongest operational feature, and the reason it scaled the early web.

```text
PRIMARY ──binlog──► REPLICA 1  (read traffic)
                └─► REPLICA 2  (read traffic)
                └─► REPLICA 3  (backups, analytics)
```

- **Asynchronous by default** — replicas lag, sometimes by seconds
- **Semi-synchronous** available when you need stronger guarantees
- **Group Replication / InnoDB Cluster** for automated failover

> [!CAUTION]
> **Read-after-write on a replica is a classic bug.** A user updates their profile, the next request reads from a lagging replica, and the change appears to have vanished. Route reads that must see recent writes to the primary.

---

## 8. Real World Example

- **WordPress** — a very large share of the web, all on MySQL.
- **Facebook** ran (and heavily modified) MySQL at enormous scale.
- **YouTube** built **Vitess** on top of MySQL to shard it horizontally; Vitess is now used well beyond YouTube.
- **PlanetScale** productised Vitess as a serverless MySQL platform.

---

## 9. Communication and Dependencies

- **Applications** through drivers and pools
- **Replicas** through the binary log
- **[ORMs](ORM.md)** — Eloquent, Hibernate, Prisma, SQLAlchemy all support it
- **ProxySQL** for connection pooling and read/write splitting

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Use MySQL when the ecosystem expects it, when read-heavy simple queries dominate, or when your team's operational experience is with MySQL replication. It is a genuinely good database, and familiarity is a real advantage.

> [!CAUTION]
> - **Complex analytical queries** — PostgreSQL's planner and window function support are stronger.
> - **Heavy JSON or document work** — JSONB is materially better.
> - **Extensions** — geospatial, vectors, time series are all weaker or absent.

---

## 11. Advantages and Disadvantages

**Advantages**
- Very fast for simple, read-heavy workloads
- Thread-per-connection is lighter than process-per-connection
- Exceptionally mature replication
- Available everywhere, including the cheapest hosting
- Huge community and abundant expertise

**Disadvantages**
- Weaker query planner for complex queries
- Historical correctness defaults that still appear in old installations
- Oracle ownership creates licensing caution (MariaDB is the response)
- Fewer extensions and advanced types
- Some legacy sharp edges around character sets and DDL

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **innodb_buffer_pool_size** | The single most important setting — typically 70–80% of RAM |
| **Indexes** | As always, the dominant factor |
| **Connections** | Threads are cheaper than processes, but pooling is still correct |
| **Replica lag** | A correctness concern as much as a performance one |

---

## 13. Security Considerations

> [!CAUTION]
> **Run `mysql_secure_installation` on any new server.** Historical defaults included anonymous users, a remote-accessible root account and a test database.

- **Parameterised queries** — the same SQL injection rules as everywhere
- **Least privilege** — the application user needs `SELECT/INSERT/UPDATE/DELETE`, not `DROP` or `GRANT`
- **Never expose port 3306** to the internet
- **TLS for connections**, especially to replicas across networks
- **`LOAD DATA LOCAL INFILE`** can be abused by a malicious server to read client files — disable it unless needed
- **Test restores**, as with any database

---

## 14. Mental Model

> [!NOTE]
> **MySQL is a reliable, extremely common van; PostgreSQL is a workshop truck.**
>
> The van is everywhere, everyone can drive one, and it does the ordinary job well. The truck carries specialised equipment and handles awkward loads better. For most deliveries, either is fine — and the fleet you already maintain matters more than the spec sheet.

---

## 15. Mini Architecture Diagram

```text
App
    ↓
ProxySQL / pool
    ↓
MySQL primary  (InnoDB: buffer pool, redo log, MVCC)
    ↓ binlog
Replicas → read traffic
```

---

## 16. Complete Request Flow

```text
Query arrives on a pooled connection (handled by a thread)
    ↓
Parser → optimiser → execution plan
    ↓
InnoDB: pages from the buffer pool, or read from disk
    ↓
MVCC returns the version visible to this transaction
    ↓
On write: redo log written first, then the page modified
    ↓
COMMIT acknowledged after the redo log is durable
    ↓
Change written to the binlog → streamed to replicas
    ↓
A read hitting a lagging replica may not see it yet ← plan for this
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> MySQL is fast, ubiquitous and operationally mature — use InnoDB and `utf8mb4`, and design for replica lag rather than assuming replicas are current.

---

## 18. Common Mistakes

- **`utf8` instead of `utf8mb4`** — emoji and some characters silently break
- **MyISAM tables** — no transactions, no crash safety
- **Reading from a replica immediately after a write**
- **`innodb_buffer_pool_size` left at the default** on a large machine
- **Application connecting as root**
- **Assuming DDL is transactional** on older versions
- **Not running `mysql_secure_installation`**

---

## 19. Open Source Technologies

- **MySQL**, **MariaDB**, **Percona Server** — the engines
- **Vitess** — horizontal sharding, from YouTube
- **ProxySQL** — pooling and read/write splitting
- **Percona Toolkit**, **pt-online-schema-change** — safe migrations on large tables
- **mysqldump**, **XtraBackup** — backups

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Check the character set of your tables — is it `utf8` or `utf8mb4`?
- [ ] Measure your replica lag under load and decide which reads must go to the primary.
- [ ] Verify `innodb_buffer_pool_size` is sized for your actual machine.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
App → pool → MySQL primary (InnoDB) → binlog → replicas
```

## 2. Request Flow

```text
Input       a SQL query on a thread-backed connection
    ↓
Processing  optimised, executed against the InnoDB buffer pool, MVCC-filtered
    ↓
Output      rows; writes durable in the redo log, then replicated asynchronously
```

## 3. Real-World Usage

**YouTube built Vitess** to shard MySQL horizontally rather than replacing it. Keeping a familiar, well-understood engine and solving scale around it is a recurring and usually sound pattern.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A widely deployed open-source relational database |
| **Why does it exist?** | To provide a free, fast, simple relational database for the web |
| **Where does it belong?** | Behind a pool, beneath your application, with replicas for reads |
| **When should I use it?** | When the ecosystem or team expects it — otherwise PostgreSQL by default |

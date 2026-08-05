# MongoDB

> **In one line —** a document database that stores JSON-like objects instead of rows; excellent when data is genuinely document-shaped, and painful when it turns out to be relational after all.

| | |
|---|---|
| **Category** | Document Database (NoSQL) |
| **Architectural Layer** | Data |
| **Data model** | BSON documents in collections |
| **Related notes** | [Database Fundamentals](Database%20Fundamentals.md) · [PostgreSQL](PostgreSQL.md) · [Transactions](Transactions.md) · [ORM](ORM.md) |

---

## 1. Short Definition

*What is it?*

MongoDB stores **documents** — nested, JSON-like structures — in collections, with no enforced schema by default. A document contains its related data inline rather than spread across joined tables.

---

## 2. Purpose

*What is its main purpose?*

To store data in the shape the application already uses, so that reading an entity is one lookup rather than a join across five tables.

---

## 3. Problem

*What engineering problem does it solve?*

```text
RELATIONAL                            DOCUMENT
users                                 {
orders                                  _id: 1,
order_items                             name: "Ana",
addresses                               orders: [
                                          { id: 7, items: [...] }
4 tables, 3 joins to read one           ],
customer with their orders              addresses: [...]
                                      }
                                      one lookup, one document
```

It also removes migration friction: adding a field is writing a document with that field, not altering a table.

---

## 4. Architecture Position

```text
Application
    ↓
Driver / Mongoose
    ↓
┌──────── MongoDB ────────┐
│  mongos (router)        │  in a sharded cluster
│  query engine           │
│  WiredTiger storage     │
│  indexes                │
└───────────┬─────────────┘
            ↓
    Replica set: primary + secondaries
```

---

## 5. Where it genuinely fits

```text
✓ Content and catalogues         products with varying attributes
✓ Event and activity logs        write-heavy, append-only
✓ User-generated content         nested, irregular structures
✓ Configuration and profiles     schema differs per record
✓ Real prototypes                schema still moving daily
```

```text
✗ Financial and transactional    needs relational integrity
✗ Highly interconnected data     "friends of friends who bought X"
✗ Reporting and analytics        aggregation pipelines are harder than SQL
✗ Data that is relational        which is most business data
```

> [!IMPORTANT]
> **The schema question does not disappear — it moves into the application.** Without a database schema, every piece of code that reads a document must handle every historical shape it might have. Teams that adopted MongoDB for "no schema" usually end up enforcing one in the application layer anyway (Mongoose, Zod, Pydantic), which is a database schema written in a less reliable place.

---

## 6. Embedding vs referencing — the central design decision

```text
EMBED                                 REFERENCE
{ order, items: [...] }               { order, item_ids: [1,2,3] }
    ↓                                     ↓
one read, atomic update               separate queries, manual joins
bounded, read-together data           unbounded or shared data
16 MB document limit                  no size limit
```

> [!CAUTION]
> **Embedding unbounded arrays is the classic MongoDB failure.** A `user` document embedding every `login_event` grows without limit, hits the 16 MB document cap, and makes every read of that user expensive. Embed what is bounded and read together; reference everything else.

---

## 7. Transactions

MongoDB gained multi-document ACID transactions in version 4.0, and they work — but they are more expensive than in a relational database and are not the natural grain of the system.

> [!TIP]
> If you find yourself using multi-document transactions frequently, the data is telling you it is relational. That is a signal about the model, not a limitation to work around.

---

## 8. Real World Example

- **Content management and product catalogues** — genuinely variable structure per item.
- **IoT and event ingestion** — high write volume, append-heavy, flexible payloads.
- **MEAN/MERN stacks** — MongoDB became the default largely through JavaScript ecosystem gravity rather than data-model fit.
- **PostgreSQL JSONB** covers a large share of document use cases while keeping relational capability, which is why the "one database" argument is strong.

---

## 9. Communication and Dependencies

- **Official drivers**, or **Mongoose** in Node (which adds the schema back)
- **Replica set** for durability and failover — a single node is not a production deployment
- **Sharding** for horizontal write scaling, keyed by a shard key
- **Atlas** as the managed offering

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Use MongoDB when documents are the natural unit — read whole, written whole, with genuinely varying structure — and when horizontal write scaling is a real requirement.

> [!CAUTION]
> Do not choose it because "schemas are annoying" or because it appeared in a tutorial stack. For most business applications the data is relational, and choosing a document store means reimplementing joins and integrity in application code — the most expensive possible place to put them.

---

## 11. Advantages and Disadvantages

**Advantages**
- Documents match application objects directly
- Flexible structure, no migration for new fields
- Horizontal scaling and sharding built in
- Strong write throughput
- Excellent developer experience for simple cases

**Disadvantages**
- No enforced schema means inconsistent data accumulates
- Joins (`$lookup`) are limited and slower than SQL joins
- Multi-document transactions are costly
- The aggregation pipeline is powerful but harder to read than SQL
- Historically weak security defaults (see below)
- Denormalisation makes updates touch many documents

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Single-document reads** | Very fast — no joins |
| **Indexes** | As critical as anywhere; unindexed queries scan the collection |
| **Document size** | Large documents make every read expensive |
| **Working set** | Should fit in RAM, as with any database |
| **Shard key** | A poor choice creates hotspots that are painful to fix later |

---

## 13. Security Considerations

> [!CAUTION]
> **MongoDB shipped for years with no authentication and binding to all interfaces.** Tens of thousands of instances were found exposed and ransomed. Modern versions default to localhost with auth required — but inherited deployments and careless container configurations still reproduce the old mistake.

- **Enable authentication and bind to a private interface** — verify this, do not assume it
- **NoSQL injection is real**: a query built from unvalidated input can accept `{"$gt": ""}` as a password and match every user. Validate input types before they reach the query
- **Field-level access** is weaker than SQL's — application code carries more of the burden
- **Enable TLS** between application and database
- **Least privilege** roles per application

---

## 14. Mental Model

> [!NOTE]
> **A relational database is a filing cabinet with a folder per topic; MongoDB is a box of complete case files.**
>
> Pulling one case file gives you everything about it at once. But if the same address appears in four hundred files and the street is renamed, you are editing four hundred files — and nothing prevents two of them from disagreeing.

---

## 15. Mini Architecture Diagram

```text
Application
    ↓
Driver
    ↓
mongos router  (sharded clusters)
    ↓
┌── shard 1 ──┐  ┌── shard 2 ──┐
│ primary      │  │ primary     │
│ secondary ×2 │  │ secondary×2 │
└──────────────┘  └─────────────┘
```

---

## 16. Complete Request Flow

```text
find({ user_id: 42, status: "pending" })
    ↓
Driver sends it to the primary (or a secondary, per read preference)
    ↓
Index on (user_id, status)?  → index scan
No index?                    → full collection scan
    ↓
Documents returned whole — no join needed
    ↓
On write: applied to the primary, written to the oplog
    ↓
Replicated to secondaries asynchronously
    ↓
Write concern decides how many must acknowledge before it is "done"
```

> [!IMPORTANT]
> **Write concern is a correctness decision.** `w: 1` acknowledges after the primary alone — a failover moments later can lose that write. `w: "majority"` is slower and durable. Defaults have changed across versions; set it explicitly.

---

## 17. Key Takeaway

> [!IMPORTANT]
> MongoDB is excellent when data is genuinely document-shaped and read whole — and if you find yourself joining, transacting across documents, and enforcing a schema in code, the data was relational.

---

## 18. Common Mistakes

- **Choosing it to avoid schemas**, then reimplementing schemas in the application
- **Embedding unbounded arrays** until documents hit the 16 MB limit
- **Missing indexes** — the same fatal mistake as in any database
- **Building queries from unvalidated input** — NoSQL injection
- **Running a single node** in production instead of a replica set
- **Choosing a shard key badly** — extremely expensive to change
- **Not setting write concern explicitly**

---

## 19. Open Source Technologies

- **MongoDB Community**, **Atlas** (managed)
- **Mongoose** — schemas and validation in Node
- **FerretDB** — MongoDB-compatible API over PostgreSQL
- **PostgreSQL JSONB** — the strongest alternative for many document workloads
- **Compass** — GUI and query profiler

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Take one of your collections and check whether any document contains an array that can grow without bound.
- [ ] Verify authentication is enabled and the instance is not bound to a public interface.
- [ ] Write down the same data model in both MongoDB and PostgreSQL, and compare which reads and writes get harder.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
App → driver → mongos → shards (primary + secondaries) → disk
```

## 2. Request Flow

```text
Input       a document query or write
    ↓
Processing  index lookup or collection scan; writes go to the primary and the oplog
    ↓
Output      whole documents; replication governed by write concern
```

## 3. Real-World Usage

**Content and catalogue systems** are MongoDB's most defensible use case: products with genuinely different attributes per category, read as complete objects. Where the data turned out to be relational, many teams migrated back to PostgreSQL with JSONB.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A document database storing nested JSON-like records |
| **Why does it exist?** | So data can be stored in the shape the application uses |
| **Where does it belong?** | As the primary store for document-shaped data |
| **When should I use it?** | When documents are read and written whole — not to avoid schemas |

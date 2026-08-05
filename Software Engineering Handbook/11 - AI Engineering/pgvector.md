# pgvector

> **In one line —** vector search as a PostgreSQL extension, which means similarity search that joins, transacts and backs up alongside the rest of your data.

| | |
|---|---|
| **Category** | PostgreSQL Extension |
| **Architectural Layer** | Data |
| **Related notes** | [Vector Databases](Vector%20Databases.md) · [PostgreSQL](../08%20-%20Databases%20and%20Data/PostgreSQL.md) · [Embeddings](Embeddings.md) · [RAG](RAG.md) · [Indexes](../08%20-%20Databases%20and%20Data/Indexes.md) |

---

## 1. Short Definition

*What is it?*

pgvector adds a `vector` column type and similarity operators to PostgreSQL, with approximate nearest-neighbour indexes. Vector search becomes ordinary SQL.

---

## 2. Purpose

*What is its main purpose?*

To make similarity search available without adding a separate database — keeping vectors, source content, permissions and business data in one system.

---

## 3. Problem

*What engineering problem does it solve?*

```text
SEPARATE VECTOR DATABASE               PGVECTOR
vectors over here                      one database
source text in PostgreSQL                  ↓
permissions in PostgreSQL              JOIN vectors to business data
    ↓                                  ONE transaction
two systems to keep in sync            existing backups cover it
two backups, two access models         existing access control applies
sync bugs → orphaned vectors           no sync to get wrong
```

> [!IMPORTANT]
> **The consistency argument is the strongest one.** With a separate vector store, deleting a document requires deleting a row *and* its vectors, in two systems, without a transaction. That gap produces orphaned vectors — which for a RAG system means retrieving content that no longer exists, or that the user is no longer permitted to see.

---

## 4. Architecture Position

```text
Application
    ↓
PostgreSQL
  ├─ documents table      (source text, permissions, business columns)
  └─ embeddings table     (vector column + HNSW index)
    ↓
One SQL query: similarity + filter + join, in one transaction
```

---

## 5. How it looks

```sql
CREATE EXTENSION vector;

CREATE TABLE chunks (
  id         bigserial PRIMARY KEY,
  doc_id     bigint REFERENCES documents(id) ON DELETE CASCADE,
  tenant_id  bigint NOT NULL,
  content    text,
  embedding  vector(768)
);

CREATE INDEX ON chunks USING hnsw (embedding vector_cosine_ops);
```

```sql
-- similarity search WITH permission filtering and a join, in one query
SELECT c.content, d.title, c.embedding <=> $1 AS distance
FROM chunks c
JOIN documents d ON d.id = c.doc_id
WHERE c.tenant_id = $2                    -- filtering the database does natively
  AND d.status = 'published'
ORDER BY c.embedding <=> $1
LIMIT 5;
```

> [!TIP]
> That single query does similarity search, permission filtering, a join and ordering. In a separate vector store, this is three round trips and application-side merging — and the filtering is the part most likely to be implemented incorrectly.

---

## 6. The operators

```text
<=>   cosine distance        ← the usual choice for text embeddings
<->   Euclidean (L2) distance
<#>   negative inner product
```

Use the one matching your embedding model's training. Getting it wrong degrades results quietly.

---

## 7. Index types

```text
HNSW    graph-based
        ✓ better recall, faster queries, no training step
        ✗ more memory, slower to build
        → the default choice

IVFFlat clustered
        ✓ smaller, faster to build
        ✗ must be built AFTER data is loaded, and rebuilt as data grows
        → use when memory is tight
```

> [!CAUTION]
> **Without an index, PostgreSQL performs an exact sequential scan** over every vector. That is *correct* but slow — and it works fine in development with a thousand rows, then collapses in production. Check that your index is actually being used with `EXPLAIN ANALYZE`.

---

## 8. Tuning

```sql
SET hnsw.ef_search = 100;   -- higher → better recall, slower
```

```text
Build-time:  m, ef_construction    → index quality
Query-time:  ef_search             → the recall/speed trade, per query
```

> [!TIP]
> `ef_search` is adjustable per session, so you can use a low value for autocomplete and a high one for a RAG retrieval where recall matters more than a few milliseconds.

---

## 9. Where the limits are

```text
Up to ~1 million vectors      pgvector is comfortable
1–10 million                  workable, with tuning and enough RAM
Beyond ~10–50 million         a dedicated vector database starts to earn its cost
```

> [!IMPORTANT]
> Most applications never reach the first threshold. A corpus of 50,000 documents chunked into 500,000 pieces sits well inside pgvector's comfortable range — and the operational simplicity of not running a second database is worth a great deal.

---

## 10. Real World Example

- **RAG over internal documentation** — the most common deployment by far.
- **Supabase and Neon** offer pgvector by default, making it the path of least resistance for many teams.
- **Recommendations** joined against inventory and pricing in the same query.
- **Deduplication** — finding near-identical records with a similarity threshold.

---

## 11. Communication and Dependencies

- **PostgreSQL** with the extension installed and enabled
- **An embedding model** producing vectors of a fixed dimension
- **Sufficient RAM** — HNSW indexes should fit in `shared_buffers` and the OS cache
- **A migration path** for changing embedding model

---

## 12. When To Use / When NOT To Use

> [!TIP]
> **Use pgvector by default.** If you already run PostgreSQL and have fewer than a few million vectors, the burden of proof should be on choosing something else.

> [!CAUTION]
> Move to a dedicated system when you have tens of millions of vectors, need distributed sharding of the index, or require specialised features such as multi-vector search. Those are real thresholds — but far higher than most teams assume when they reach for a vector database on day one.

---

## 13. Advantages and Disadvantages

**Advantages**
- **No new infrastructure**
- Transactional consistency between vectors and source data
- SQL joins and filters combined with similarity in one query
- Existing backups, replication, monitoring and access control apply
- Row-level security works on vector tables

**Disadvantages**
- Scales less far than dedicated systems
- HNSW index build is slow on large tables
- Vector operations consume PostgreSQL's memory budget, competing with regular queries
- Fewer specialised features
- The extension must be available — some managed providers lag

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Query with HNSW** | Single-digit milliseconds up to ~1M vectors |
| **Without an index** | Sequential scan — fine at 1,000 rows, unusable at 1,000,000 |
| **Memory** | The index competes with your normal working set |
| **Index build** | Minutes to hours on large tables |
| **Storage** | A 768-dimension vector is ~3 KB per row |

> [!TIP]
> **Use `CREATE INDEX CONCURRENTLY`** on a live table, and consider a **half-precision** vector type if available in your version — it halves storage and memory with little accuracy loss.

---

## 15. Security Considerations

> [!IMPORTANT]
> This is pgvector's strongest and least-discussed advantage: **PostgreSQL's Row-Level Security applies to vector tables**. Multi-tenant isolation can be enforced by the database itself rather than by every query remembering a `WHERE tenant_id = ...`.

```sql
ALTER TABLE chunks ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON chunks
  USING (tenant_id = current_setting('app.tenant_id')::bigint);
```

- **`ON DELETE CASCADE` removes vectors** when the source document is deleted — no orphans, no separate cleanup job
- **Embeddings are reconstructible** and inherit the sensitivity of their source; see [Embeddings](Embeddings.md)
- **Existing database encryption, auditing and least-privilege roles** all apply without additional work

---

## 16. Mental Model

> [!NOTE]
> **pgvector is a new kind of index on a database you already run.**
>
> You did not deploy a separate system when you needed full-text search or JSON queries — you used an index. This is the same move: similarity becomes another thing PostgreSQL can order by.

---

## 17. Mini Architecture Diagram

```text
Application
    ↓
PostgreSQL
  documents ──1:N── chunks (vector + HNSW index)
       ↑                ↑
   permissions      RLS policy
    ↓
One query: similarity ORDER BY + WHERE filter + JOIN
```

---

## 18. Complete Request Flow

```text
─────────── ingestion ───────────
Document uploaded
    ↓
BEGIN
    INSERT into documents
    chunks embedded and INSERTed with the document's id
COMMIT                             ← atomic; no orphaned vectors possible
    ↓
─────────── query ───────────
User question embedded
    ↓
One SQL query:
    ORDER BY embedding <=> $1
    WHERE tenant_id = current tenant   ← enforced by RLS, not by memory
    JOIN documents for the title and URL
    LIMIT 5
    ↓
~4 ms; results already permission-filtered
    ↓
Passed to the LLM as grounding context
    ↓
─────────── deletion ───────────
DELETE FROM documents WHERE id = 42
    ↓
ON DELETE CASCADE removes the chunks and their vectors
    ↓
Nothing to synchronise, nothing to forget
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> pgvector gives you vector search with transactions, joins and row-level security in a database you already operate — which is why it should be the default until scale genuinely forces something else.

---

## 20. Common Mistakes

- **No index created**, so PostgreSQL sequentially scans every vector
- **Wrong distance operator** for the embedding model
- **Adding a dedicated vector database** before measuring whether pgvector suffices
- **Not using RLS** for multi-tenant isolation, then filtering manually in every query
- **`ef_search` left at the default** when recall matters
- **Building an index on a live table** without `CONCURRENTLY`
- **No `ON DELETE CASCADE`**, producing orphaned vectors after all

---

## 21. Open Source Technologies

- **pgvector** — the extension
- **pgvectorscale** — additional index types for larger datasets
- **Supabase**, **Neon** — managed PostgreSQL with pgvector enabled
- **sentence-transformers** — generating the embeddings
- **Qdrant**, **Milvus** — the alternatives, when scale demands them

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Run `EXPLAIN ANALYZE` on a similarity query and confirm the HNSW index is used.
- [ ] Set up row-level security on a vector table and verify a second tenant cannot retrieve its rows.
- [ ] Compare query latency at `ef_search` 40 and 200, and measure the recall difference.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Application → PostgreSQL (documents + vector chunks + HNSW + RLS)
```

## 2. Request Flow

```text
Input       a query vector, plus filters and joins
    ↓
Processing  one SQL statement combining similarity, filtering and joining
    ↓
Output      permission-filtered results, transactionally consistent with the source
```

## 3. Real-World Usage

**Supabase and Neon** enable pgvector by default, which has made "just use Postgres" the practical default for RAG in many teams. The reduction from two databases to one removes an entire class of synchronisation bug.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A PostgreSQL extension adding vector types and similarity indexes |
| **Why does it exist?** | So similarity search does not require a second database |
| **Where does it belong?** | Inside the database that already holds your content |
| **When should I use it?** | By default — until vector count genuinely exceeds what it handles |

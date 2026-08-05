# Qdrant

> **In one line —** a dedicated vector database whose distinguishing feature is that filtering happens *during* the search, not after it.

| | |
|---|---|
| **Category** | Vector Database |
| **Architectural Layer** | Data |
| **Written in** | Rust |
| **Related notes** | [Vector Databases](Vector%20Databases.md) · [pgvector](pgvector.md) · [Weaviate](Weaviate.md) · [Milvus](Milvus.md) · [RAG](RAG.md) |

---

## 1. Short Definition

*What is it?*

Qdrant is an open-source vector database built for similarity search with rich metadata filtering. Vectors are stored as **points** with a payload of arbitrary JSON, and filters are applied inside the index traversal.

---

## 2. Purpose

*What is its main purpose?*

To answer "find similar items that also match these conditions" quickly and correctly — which is the query almost every real application actually needs.

---

## 3. Problem

*What engineering problem does it solve?*

```text
NAIVE FILTERING (post-filter)
    search → top 100 nearest
    ↓
    filter by tenant / permission / date
    ↓
    2 results survive — or none at all
    ↓
    you asked for 10 and got 0, with no error

QDRANT (filterable index)
    filter conditions are applied DURING graph traversal
    ↓
    you get 10 results that match both similarity AND the filter
```

> [!IMPORTANT]
> This sounds like a detail and is not. **Permission filtering is a correctness and security requirement**, and post-filtering silently returns wrong or empty results precisely when the filter is selective — which is exactly the multi-tenant case.

---

## 4. Architecture Position

```text
Application
    ↓  gRPC or REST
┌──────── QDRANT ────────┐
│  collections            │
│  points: vector + payload
│  HNSW index (filterable)│
│  payload indexes        │
└───────────┬─────────────┘
            ↓
     Disk / memory-mapped storage
```

---

## 5. The data model

```text
COLLECTION       a set of points with the same vector dimension
POINT            id + vector(s) + payload (arbitrary JSON)
PAYLOAD INDEX    an index on a payload field, for fast filtering
```

```json
{
  "id": 42,
  "vector": [0.21, -0.45, ...],
  "payload": {
    "tenant_id": 7,
    "doc_id": 1001,
    "published": true,
    "created": "2026-08-05"
  }
}
```

> [!TIP]
> **Create payload indexes on every field you filter by.** Without them, filtering still works but degrades to scanning — which removes the main reason for choosing Qdrant.

---

## 6. Features worth knowing

| Feature | Use |
|---|---|
| **Filterable HNSW** | Filtering inside the search, not after |
| **Quantisation** | Scalar, product and binary — large memory savings |
| **Multi-vector points** | Several embeddings per item (title, body, image) |
| **Sparse vectors** | Keyword-style scoring alongside dense — hybrid search |
| **Snapshots** | Backup and restore of collections |
| **Distributed mode** | Sharding and replication |

---

## 7. Quantisation — the memory lever

```text
float32 vectors, 768 dimensions, 5M points
    ↓
~15 GB of RAM for the raw vectors
    ↓ scalar quantisation (int8)
~4 GB, with a small accuracy loss
    ↓ binary quantisation
~0.5 GB, larger accuracy loss, rescoring recommended
```

> [!TIP]
> Quantisation with **rescoring** — search the compressed vectors, then re-rank the top candidates against the originals — usually recovers most of the lost accuracy while keeping the memory saving. It is the standard configuration at scale.

---

## 8. Real World Example

- **Multi-tenant RAG** — the case Qdrant fits best, because every query must filter by tenant.
- **E-commerce similarity search** — "similar products, in stock, in this category, under €500".
- **Recommendation systems** with strong metadata constraints.
- **Hybrid search** using dense and sparse vectors together in one query.

---

## 9. Qdrant vs the alternatives

| | Qdrant | [pgvector](pgvector.md) | [Weaviate](Weaviate.md) | [Milvus](Milvus.md) |
|---|---|---|---|---|
| **New infrastructure** | Yes | **No** | Yes | Yes |
| **Filtering** | **Excellent** | Native SQL | Good | Good |
| **Transactions with your data** | No | **Yes** | No | No |
| **Scale ceiling** | High | Moderate | High | **Very high** |
| **Operational weight** | Light | **None extra** | Moderate | Heavy |

> [!IMPORTANT]
> **Qdrant's honest position is: the dedicated option you reach for when pgvector is no longer enough.** Its filtering is genuinely better than most, its operational footprint is modest, and its defaults are sensible. But it is still a second database — with its own backups, access control and consistency gap with your primary store.

---

## 10. Communication and Dependencies

- **REST or gRPC** clients — gRPC is noticeably faster for bulk operations
- **An embedding model** — Qdrant stores vectors, it does not generate them
- **RAM** for the index, unless using on-disk mode with quantisation
- **Your primary database** for source content and permissions

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Qdrant when you have outgrown pgvector, when filtering is central to your queries, or when you need multi-tenant isolation with strong performance.

> [!CAUTION]
> - **Not before measuring whether pgvector suffices** — below a million vectors it usually does
> - **Not as a source of truth** — keep source content in your primary database
> - **Not without a re-indexing plan** for embedding model changes
> - **Not without accounting for the consistency gap** — a deleted document must have its vectors deleted separately, and that is not transactional

---

## 12. Advantages and Disadvantages

**Advantages**
- Filtering applied during search, correctly
- Rust — fast, memory-efficient, no garbage collection pauses
- Sensible defaults; straightforward to run
- Quantisation options for large collections
- Sparse vectors enable hybrid search in one system
- Good client libraries and clear documentation

**Disadvantages**
- A second database to operate, back up and secure
- No transactional consistency with your primary data
- Re-indexing required on embedding model change
- Smaller ecosystem than PostgreSQL
- Distributed mode adds real operational complexity

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Query** | Single-digit milliseconds on millions of points |
| **Memory** | The main cost; quantisation is the primary lever |
| **Filtered queries** | Fast **only with payload indexes** |
| **Bulk upload** | Use batching and gRPC; disable indexing during large loads |
| **Rescoring** | Small extra cost, recovers accuracy after quantisation |

---

## 14. Security Considerations

> [!CAUTION]
> **Qdrant ships without authentication by default.** Deployed on a network reachable by anything untrusted, it is fully readable and writable — repeating the pattern that made exposed Redis, MongoDB and Elasticsearch instances a recurring news story.

- **Set an API key and enable TLS** before it leaves localhost
- **Never expose it to the internet**, and never to a browser
- **Store tenant and permission fields in the payload**, index them, and filter on every query — this is the primary access control
- **Vectors are reconstructible** and inherit the sensitivity of their source
- **Deletion must be propagated** from your primary database; a missed deletion means retrievable content the user should no longer see
- **Back up collections** — snapshots exist, but they are not automatic

---

## 15. Mental Model

> [!NOTE]
> **Most vector search is a librarian who fetches the ten nearest books and then checks whether you are allowed to read them — sometimes handing you none.**
>
> Qdrant checks your permissions while walking the shelves, so the ten books it brings back are ten you can actually read.

---

## 16. Mini Architecture Diagram

```text
Primary database  (source of truth: documents, permissions)
    ↓  embed
┌──────── QDRANT ────────┐
│ collection             │
│  point = vector + payload
│  payload indexes: tenant_id, status
│  filterable HNSW       │
└───────────┬─────────────┘
            ↓
Query: vector + filter (tenant_id = 7, status = published)
            ↓
Top-k, already permitted → rerank → LLM
```

---

## 17. Complete Request Flow

```text
─────────── indexing ───────────
Document chunked and embedded
    ↓
Upserted as points with payload: tenant_id, doc_id, status, created
    ↓
Payload indexes exist on tenant_id and status
    ↓
─────────── query ───────────
User in tenant 7 asks a question
    ↓
Query embedded
    ↓
Qdrant search with filter: tenant_id = 7 AND status = "published"
    ↓
Filter applied DURING HNSW traversal
    ↓
10 relevant, permitted results returned in ~3 ms
    ↓
Reranked; source text fetched from the primary database by doc_id
    ↓
─────────── deletion ───────────
Document deleted in PostgreSQL
    ↓
An explicit delete-by-filter call removes its points in Qdrant
    ↓  ← this step is not transactional, and forgetting it leaks content
```

> [!CAUTION]
> That final gap is the structural cost of any separate vector store. Make deletion propagation an explicit, monitored part of the pipeline — not something a developer must remember.

---

## 18. Key Takeaway

> [!IMPORTANT]
> Qdrant's advantage is filtering during search rather than after it, which is what multi-tenant retrieval actually requires — but it is a second database, with a consistency gap that pgvector does not have.

---

## 19. Common Mistakes

- **No payload indexes**, so filtering degrades to scanning
- **Running without authentication**
- **Choosing it before measuring pgvector**
- **Forgetting to propagate deletions**, leaving retrievable orphaned content
- **No permission field in the payload** — relying on application-side filtering afterwards
- **Loading millions of points without batching** or disabling indexing during the load
- **Treating it as the source of truth**

---

## 20. Open Source Technologies

- **Qdrant** — server and clients, with a managed cloud option
- **pgvector** — the option to rule out first
- **Weaviate**, **Milvus**, **Chroma** — alternatives
- **FastEmbed** — Qdrant's lightweight embedding library
- **Rerankers** — cross-encoders that improve final ordering

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Compare filtered query latency with and without a payload index on the filter field.
- [ ] Verify authentication is enabled and the instance is not reachable from outside your network.
- [ ] Trace what happens in your system when a document is deleted — do its vectors go too?

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Primary DB → embeddings → Qdrant (points + payload + filterable HNSW) → rerank
```

## 2. Request Flow

```text
Input       a query vector plus payload filters
    ↓
Processing  filters applied during HNSW traversal, not afterwards
    ↓
Output      top-k results that satisfy both similarity and permission
```

## 3. Real-World Usage

**Multi-tenant RAG** is Qdrant's clearest fit: every query must be scoped to one tenant, and post-filtering approaches either return too few results or leak across tenants. Filtering inside the index is what makes that query both correct and fast.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An open-source vector database with filtering inside the index |
| **Why does it exist?** | Because post-filtering returns wrong results on selective queries |
| **Where does it belong?** | Beside your primary database, as a search index |
| **When should I use it?** | When pgvector is outgrown and filtering is central |

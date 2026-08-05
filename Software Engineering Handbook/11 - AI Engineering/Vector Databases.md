# Vector Databases

> **In one line —** databases that find the nearest vectors to a query vector quickly, by giving up the guarantee that they found the actual nearest ones.

| | |
|---|---|
| **Category** | Overview note *(hub for this sub-section)* |
| **Architectural Layer** | Data |
| **Sub-topics** | [Qdrant](Qdrant.md) · [Weaviate](Weaviate.md) · [Milvus](Milvus.md) · [pgvector](pgvector.md) |
| **Related notes** | [Embeddings](Embeddings.md) · [RAG](RAG.md) · [Database Fundamentals](../08%20-%20Databases%20and%20Data/Database%20Fundamentals.md) |

---

## 1. Short Definition

*What is it?*

A vector database stores high-dimensional vectors and answers **nearest-neighbour queries**: given this vector, return the most similar stored ones, fast.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Exact nearest-neighbour search over 10 million vectors
    ↓
Compare the query against every one of them
    ↓
10 million distance calculations per query
    ↓
Far too slow
```

The solution is **approximate** nearest neighbour search — accept a small chance of missing a true nearest neighbour in exchange for orders of magnitude more speed.

> [!IMPORTANT]
> **This is the defining trade of the entire category.** A vector database does not guarantee correct results. It typically returns 95–99% of the true nearest neighbours, and that is considered success. Anyone expecting database-style exactness will be surprised.

---

## 3. Architecture Position

```text
Documents → chunks → embedding model
    ↓
VECTOR DATABASE
    vectors + metadata + index (HNSW / IVF)
    ↓
Query vector → approximate nearest neighbours
    ↓
Filtered by metadata and permissions → reranked → LLM or UI
```

---

## 4. How approximate search works

```text
HNSW  (hierarchical navigable small world)
    a layered graph; search starts coarse and refines
    → the most common choice; fast, accurate, memory-hungry

IVF   (inverted file)
    vectors clustered; search only the nearest clusters
    → less memory, needs training on the data

PRODUCT QUANTISATION
    compress vectors into codes
    → large memory savings, some accuracy loss
```

```text
Tuning knob everywhere:
    more search effort → better recall, slower
    less search effort → faster, more misses
```

---

## 5. The critical feature: metadata filtering

```text
"Find similar documents"                    → pure vector search
"Find similar documents THIS USER MAY READ" → vector search + filter
```

> [!CAUTION]
> **Filtering is where vector databases differ most, and it is not a minor feature.** Naive implementations search first and filter afterwards — which can return nothing at all if none of the top 100 results pass the filter. Proper implementations filter *during* the graph traversal.
>
> This matters enormously in practice, because **permission filtering is a correctness and security requirement**, not an optimisation.

---

## 6. Choosing one

| | Best for | Note |
|---|---|---|
| **[pgvector](pgvector.md)** | You already run PostgreSQL | **Start here.** No new system |
| **[Qdrant](Qdrant.md)** | Dedicated, strong filtering | Rust; good defaults |
| **[Weaviate](Weaviate.md)** | Built-in embedding, hybrid search | More opinionated |
| **[Milvus](Milvus.md)** | Very large scale, billions of vectors | Heavy operationally |
| **Elasticsearch / OpenSearch** | Already deployed for keyword search | Hybrid in one system |
| **FAISS** | A library, not a database | In-process, no persistence layer |

> [!TIP]
> **The honest default is pgvector.** Below roughly a million vectors, a PostgreSQL extension gives you vector search with transactions, joins, existing backups, existing access control and no new infrastructure. Dedicated vector databases earn their operational cost at larger scale or with demanding filtering requirements.

---

## 7. What they are not

```text
✗ NOT a replacement for your primary database
✗ NOT good at exact lookups
✗ NOT transactional, in most cases
✗ NOT a source of truth
```

> [!IMPORTANT]
> **Keep the source text and metadata in your primary database.** The vector store is an index that can be rebuilt. Treating it as the system of record means a corrupted index is data loss rather than an inconvenience.

---

## 8. Real World Example

- **[RAG](RAG.md) systems** — the dominant use by a wide margin.
- **Semantic search** over documentation and support content.
- **Recommendations** — "similar items" by embedding proximity.
- **Deduplication** — finding near-identical records at scale.
- **Image search** — visual similarity via image embeddings.

---

## 9. Communication and Dependencies

- **An embedding model** — and the database must be re-indexed if it changes
- **Your primary database** for source content and permissions
- **Memory** — HNSW indexes are largely RAM-resident
- **A rebuild pipeline**, because you will need it

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Use a vector database when you need similarity search over more content than a linear scan can handle — practically, above a few tens of thousands of vectors.

> [!CAUTION]
> - **Not for exact matching or attribute filtering alone** — that is a normal database
> - **Not for a few thousand vectors** — an in-memory linear scan is simpler and exact
> - **Not as your primary store**
> - **Not without a plan for re-indexing** when the embedding model changes

---

## 11. Advantages and Disadvantages

**Advantages**
- Sub-millisecond similarity search over millions of vectors
- Metadata filtering combined with similarity
- Horizontal scaling in the dedicated systems
- Purpose-built indexing and tuning

**Disadvantages**
- **Approximate — no guarantee of correct results**
- High memory requirements
- Another system to run, secure and back up
- Full re-index required on embedding model change
- Immature operationally compared with relational databases
- Rapidly changing product landscape

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Query** | Sub-millisecond to low milliseconds |
| **Memory** | HNSW typically needs the index in RAM — plan for GBs |
| **Indexing** | Building an index over millions of vectors takes real time |
| **Recall vs speed** | A direct trade, tuned per query |
| **Quantisation** | Several-fold memory reduction for modest accuracy loss |

> [!TIP]
> **Measure recall, not just latency.** A vector database returning results in 2 ms while missing a third of the relevant documents is worse than one taking 20 ms and finding them. Recall is invisible unless you build a test set and measure it.

---

## 13. Security Considerations

> [!CAUTION]
> **Permission filtering must happen inside the search, not after it.** Retrieving the top 100 results and then removing those the user cannot see is both a correctness bug — you may return nothing — and a performance problem. Worse, systems that skip filtering entirely leak content across users, and this is a common defect in hastily built RAG applications.

- **Embeddings are reconstructible** — a vector store of sensitive documents is sensitive; see [Embeddings](Embeddings.md)
- **Store permissions as metadata** and filter on them at query time
- **Deletion must remove vectors**, not only the source row
- **Authenticate the database** — many are deployed on an internal network with no authentication, repeating the Redis and Elasticsearch history
- **Poisoned entries** influence retrieval for everyone

---

## 14. Mental Model

> [!NOTE]
> **A vector database is a librarian who knows roughly where every book sits by topic.**
>
> Ask for "books like this one" and they walk you to the right shelf immediately. They may miss one that was misfiled — that is the approximation. A traditional index would find every match exactly, but only if you knew the precise title.

---

## 15. Mini Architecture Diagram

```text
Source content (primary database — the system of record)
    ↓
Chunk → embed
    ↓
┌────── VECTOR DATABASE ──────┐
│  vectors                     │
│  metadata (source, ACL, ver) │
│  HNSW index (in RAM)         │
└──────────────┬───────────────┘
               ↓
Query vector + permission filter
               ↓
Approximate nearest neighbours → rerank → application
```

---

## 16. Complete Request Flow

```text
User asks a question
    ↓
Query embedded with the SAME model used for indexing
    ↓
Vector search, WITH the permission filter applied during traversal
    ↓
Top 20 candidates returned in ~3 ms
    ↓
Reranked by a cross-encoder → top 3
    ↓
Source text fetched from the PRIMARY database by id
    ↓
Passed to the LLM as grounding context
    ↓
─────────── maintenance ───────────
A document is deleted
    ↓
Deleted from the primary database AND its vectors removed
    ↓
Embedding model upgraded
    ↓
New index built in parallel; traffic switched only when complete
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> Vector databases trade exactness for speed — start with pgvector if you already run PostgreSQL, filter permissions inside the query, and measure recall rather than only latency.

---

## 18. Common Mistakes

- **Adding a dedicated vector database** when pgvector would do
- **Filtering after search** instead of during it
- **No permission filtering at all** — a cross-user data leak
- **Treating it as the source of truth**
- **Measuring latency but never recall**
- **No plan for re-indexing** on model change
- **Deleting source rows** without deleting vectors
- **Unauthenticated deployment** on an internal network

---

## 19. Open Source Technologies

- **pgvector** — PostgreSQL extension; the sensible default
- **Qdrant**, **Weaviate**, **Milvus**, **Chroma** — dedicated
- **FAISS**, **hnswlib** — libraries for in-process search
- **Elasticsearch / OpenSearch** — vector plus keyword in one system
- **Rerankers** — cross-encoder models that substantially improve final ordering

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Build a small test set of queries with known correct answers and measure your recall.
- [ ] Check whether your permission filter runs inside the vector search or after it.
- [ ] Estimate whether pgvector would handle your corpus size before adding a new system.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Primary DB → chunks → embeddings → vector index → filtered search → rerank
```

## 2. Request Flow

```text
Input       a query vector plus metadata filters
    ↓
Processing  approximate graph traversal with filtering applied during search
    ↓
Output      the most similar permitted vectors, usually but not always correct
```

## 3. Real-World Usage

**RAG systems** are what drove this category's growth. The architectural lesson from production deployments is consistent: the vector store is an index, the primary database remains the system of record, and permission filtering belongs inside the query.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A store optimised for approximate nearest-neighbour search over vectors |
| **Why does it exist?** | Because exact similarity search does not scale |
| **Where does it belong?** | As an index beside your primary database, not in place of it |
| **When should I use it?** | Similarity search at scale — starting with pgvector |

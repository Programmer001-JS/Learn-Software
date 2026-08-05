# Weaviate

> **In one line —** a vector database that also generates the embeddings and runs hybrid search for you — convenient when that suits you, constraining when it does not.

| | |
|---|---|
| **Category** | Vector Database |
| **Architectural Layer** | Data |
| **Written in** | Go |
| **Related notes** | [Vector Databases](Vector%20Databases.md) · [Qdrant](Qdrant.md) · [pgvector](pgvector.md) · [Embeddings](Embeddings.md) · [RAG](RAG.md) |

---

## 1. Short Definition

*What is it?*

Weaviate is an open-source vector database organised around a **schema** of object classes. Its distinguishing feature is **modules**: it can call an embedding model itself, so you send text and it stores vectors.

---

## 2. Purpose

*What is its main purpose?*

To reduce the amount of pipeline you build. Weaviate handles vectorisation, hybrid search and reranking inside the database, rather than leaving each as a step in your application.

---

## 3. Problem

*What engineering problem does it solve?*

```text
TYPICAL PIPELINE                      WEAVIATE WITH A VECTORISER MODULE
chunk text                            chunk text
call an embedding model               insert the text
store vector + payload                    ↓
    ↓                                 the database embeds it
query: embed the query,               query: send text; it embeds and searches
       then search
    ↓
several moving parts to keep in sync  one call
```

---

## 4. Architecture Position

```text
Application
    ↓  GraphQL or REST
┌──────── WEAVIATE ────────┐
│  schema: classes + props  │
│  vectoriser module        │  ← optionally calls an embedding model
│  HNSW index               │
│  BM25 keyword index       │  ← hybrid search built in
│  reranker module          │
└────────────┬──────────────┘
             ↓
          Storage
```

---

## 5. Hybrid search, built in

This is Weaviate's most practically valuable feature.

```text
alpha = 1.0   pure vector search
alpha = 0.5   balanced
alpha = 0.0   pure BM25 keyword search
```

> [!TIP]
> As covered in [Embeddings](Embeddings.md), **hybrid search beats either method alone** in most production systems. Weaviate providing it as a single parameter — rather than requiring you to run two searches and merge them with reciprocal rank fusion — removes a genuine piece of work.

---

## 6. The schema model

```text
CLASS         like a table: "Article", "Product"
PROPERTIES    typed fields, filterable
CROSS-REFS    relationships between classes
VECTORISER    which module embeds this class, or "none"
```

Unlike Qdrant's schemaless payloads, Weaviate wants structure declared up front. That gives better validation and query ergonomics, at the cost of flexibility.

---

## 7. The modules trade-off

> [!IMPORTANT]
> Letting the database call the embedding model is convenient and creates coupling. Consider both directions honestly:

```text
CONVENIENT                            CONSTRAINING
one call instead of three             the database now depends on a model provider
consistent embedding of queries       an API key lives in the database config
   and documents — no drift           harder to batch, cache or control cost
less pipeline code                    changing model means reconfiguring the DB
```

> [!TIP]
> **You can set the vectoriser to `none` and supply vectors yourself.** For production systems where you want control over batching, caching and cost, that is often the better choice — and you keep the hybrid search and reranking.

---

## 8. Real World Example

- **Semantic search over content catalogues** — articles, products, documentation.
- **RAG applications** where hybrid search matters and the team wants less pipeline code.
- **Prototypes** — the built-in vectoriser makes a working search demo very quick.
- **Multi-tenancy** is supported natively, with per-tenant isolation.

---

## 9. Weaviate vs the alternatives

| | Weaviate | [Qdrant](Qdrant.md) | [pgvector](pgvector.md) |
|---|---|---|---|
| **Built-in embedding** | **Yes** | No | No |
| **Hybrid search** | **Built in** | Sparse vectors | Combine with tsvector |
| **Schema** | Required | Schemaless payload | SQL schema |
| **Filtering** | Good | **Excellent** | Native SQL |
| **New infrastructure** | Yes | Yes | **No** |
| **Query language** | GraphQL / REST | REST / gRPC | SQL |

---

## 10. Communication and Dependencies

- **A client library**, or GraphQL directly
- **An embedding provider**, if using a vectoriser module — including its API key and availability
- **RAM** for the HNSW index
- **Your primary database** for source content and permissions

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Weaviate when hybrid search is important, when you want less pipeline code, or when a schema-first model fits your data.

> [!CAUTION]
> - **Not before evaluating pgvector** — the same argument as always applies
> - **Not if you want tight control** over embedding batching and cost — or set the vectoriser to `none`
> - **Not as a source of truth**
> - **GraphQL is a genuine consideration** — pleasant for some teams, an unwanted extra concept for others

---

## 12. Advantages and Disadvantages

**Advantages**
- Hybrid search as a first-class, one-parameter feature
- Optional built-in vectorisation and reranking
- Schema validation catches data errors early
- Native multi-tenancy
- Good documentation and an active community

**Disadvantages**
- A second database, with the usual consistency gap
- Module coupling to an external embedding provider
- Schema rigidity compared with schemaless payloads
- GraphQL adds a concept if the team does not already use it
- Memory-hungry, like all HNSW-based systems
- Re-indexing required on embedding model change

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Query** | Single-digit milliseconds at typical scales |
| **Memory** | HNSW index in RAM; quantisation available |
| **Vectoriser module** | Adds the embedding provider's latency to every insert and query |
| **Hybrid search** | Slightly more expensive than pure vector — usually worth it |
| **Reranking** | Meaningful latency cost, meaningful quality gain |

> [!CAUTION]
> If the vectoriser module is enabled, **the embedding provider's availability becomes your database's availability**. An outage there means you cannot insert or query. Supplying vectors yourself removes that coupling.

---

## 14. Security Considerations

- **Enable authentication** — like its peers, Weaviate can be deployed openly and frequently is
- **API keys for embedding modules live in the database configuration** — that is a secret in an unusual place; treat it accordingly
- **Use native multi-tenancy or filter on a tenant property**, on every query
- **Vectors are reconstructible** and carry their source's sensitivity
- **Propagate deletions** from your primary database — the consistency gap is the same as any separate store
- **With a vectoriser module, your content is sent to the embedding provider** on every insert and query

---

## 15. Mental Model

> [!NOTE]
> **Qdrant is a filing system; Weaviate is a filing system with a clerk who also reads and indexes the documents for you.**
>
> The clerk saves you work and now sits between you and your files. If the clerk is off sick — the embedding provider is down — nothing can be filed or found.

---

## 16. Mini Architecture Diagram

```text
Application
    ↓ text (or text + your own vectors)
┌──── WEAVIATE ────┐
│ schema class      │
│ vectoriser module │ ──► embedding provider (optional coupling)
│ HNSW + BM25       │
│ reranker          │
└─────────┬─────────┘
          ↓
hybrid query (alpha) + filters + tenant → top-k
```

---

## 17. Complete Request Flow

```text
─────────── indexing ───────────
Article inserted as an object of class "Article"
    ↓
Vectoriser module embeds the configured properties
    ↓
Stored: vector + properties + BM25 keyword index
    ↓
─────────── query ───────────
"how do I cancel my subscription"
    ↓
Hybrid search, alpha = 0.5
    ↓
Vector search finds semantically related passages
BM25 finds exact term matches
    ↓
Scores fused
    ↓
Filtered by tenant and status
    ↓
Reranker module reorders the top 20 → top 3
    ↓
Source text fetched from the primary database
    ↓
Passed to the LLM
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Weaviate's real advantage is built-in hybrid search; its built-in vectorisation is convenient in prototypes and a coupling you may want to remove in production.

---

## 19. Common Mistakes

- **Choosing it before evaluating pgvector**
- **Leaving the vectoriser module enabled** in production, coupling availability to a provider
- **No authentication**
- **Ignoring hybrid search** and using pure vector, then wondering why exact titles do not match
- **Not propagating deletions**
- **Treating it as the system of record**
- **Schema changes underestimated** — they are more involved than adding a payload field

---

## 20. Open Source Technologies

- **Weaviate** — server, clients, and a managed cloud offering
- **pgvector**, **Qdrant**, **Milvus** — the alternatives
- **sentence-transformers** — for supplying your own vectors
- **BM25** — the keyword half of hybrid search
- **Cross-encoder rerankers** — available as a Weaviate module or run separately

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Compare results at alpha 1.0, 0.5 and 0.0 on the same query set.
- [ ] Decide whether the vectoriser module belongs in your production configuration, and write down why.
- [ ] Verify authentication is enabled and tenant filtering is applied on every query.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
App → Weaviate (schema + vectoriser + HNSW + BM25 + reranker) → results
```

## 2. Request Flow

```text
Input       text or vectors, plus filters and a hybrid alpha
    ↓
Processing  vector and keyword search fused, filtered, reranked
    ↓
Output      ranked, permitted results
```

## 3. Real-World Usage

**Hybrid search** is Weaviate's strongest practical argument. Systems that combine semantic and keyword retrieval consistently outperform either alone, and having it as one parameter rather than a hand-built fusion step is a real reduction in work.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A schema-based vector database with built-in hybrid search and vectorisation |
| **Why does it exist?** | To reduce the amount of retrieval pipeline you build yourself |
| **Where does it belong?** | Beside your primary database, as a search layer |
| **When should I use it?** | When hybrid search matters and pgvector has been ruled out |

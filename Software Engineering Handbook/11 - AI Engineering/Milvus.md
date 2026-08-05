# Milvus

> **In one line —** the vector database built for billions of vectors, with a distributed architecture that is genuinely powerful and genuinely heavy to operate.

| | |
|---|---|
| **Category** | Vector Database |
| **Architectural Layer** | Data |
| **Written in** | Go and C++ |
| **Governance** | LF AI & Data Foundation |
| **Related notes** | [Vector Databases](Vector%20Databases.md) · [Qdrant](Qdrant.md) · [pgvector](pgvector.md) · [Kubernetes](../13%20-%20DevOps%20and%20Delivery/Kubernetes.md) |

---

## 1. Short Definition

*What is it?*

Milvus is an open-source vector database designed for very large-scale similarity search, with a disaggregated architecture that separates storage, indexing, querying and coordination into independently scalable components.

---

## 2. Purpose

*What is its main purpose?*

To handle vector collections far beyond what a single node can hold — billions of vectors — with horizontal scaling of each subsystem independently.

---

## 3. Problem

*What engineering problem does it solve?*

```text
SINGLE-NODE VECTOR DATABASE
index must fit in one machine's RAM
    ↓
100 million × 768-dim float32 ≈ 300 GB
    ↓
no single machine, and no simple way to shard

MILVUS
storage, indexing and querying scale separately
    ↓
data on object storage; query nodes hold what they need
    ↓
billions of vectors, across a cluster
```

---

## 4. Architecture Position

```text
Application
    ↓  SDK
┌──────────── MILVUS ────────────┐
│  Coordinators   (root, query, data, index)
│  Proxy          (entry point)
│  Query nodes    (search)
│  Data nodes     (ingestion)
│  Index nodes    (index building)
└─────────────┬───────────────────┘
              ↓
  etcd (metadata) · object storage (S3/MinIO) · message queue (Pulsar/Kafka)
```

> [!CAUTION]
> **Look at that dependency list.** A production Milvus cluster requires etcd, object storage and a message queue in addition to Milvus itself. That is the honest cost of the architecture, and it is why Milvus is not a reasonable starting point for a system with a hundred thousand vectors.

---

## 5. The deployment options

```text
MILVUS LITE       embedded, in-process, for prototyping
MILVUS STANDALONE single node, Docker Compose — plus etcd and MinIO
MILVUS DISTRIBUTED full cluster on Kubernetes — the real product
ZILLIZ CLOUD      managed, from the same team
```

> [!TIP]
> **Milvus Lite exists precisely because the full architecture is too heavy for evaluation.** Use it to learn the API, then decide honestly whether you need the distributed version — or whether pgvector would carry you for the next two years.

---

## 6. Index types

Milvus supports more index types than most, which matters at scale.

```text
FLAT              exact search — correct, slow, small datasets
IVF_FLAT          clustered, exact within clusters
IVF_SQ8           quantised — 4× smaller
IVF_PQ            product quantisation — much smaller, more accuracy loss
HNSW              graph-based — fast and accurate, memory-hungry
DISKANN           disk-based — large datasets without holding everything in RAM
GPU indexes       GPU-accelerated build and search
```

> [!TIP]
> **DiskANN is the distinguishing capability.** It allows collections far larger than available RAM by keeping most of the index on SSD. For a billion-vector collection, that is the difference between possible and not.

---

## 7. Consistency levels

Unusually for a vector database, Milvus exposes an explicit consistency setting.

```text
STRONG        reads see all prior writes — slowest
BOUNDED       reads may lag by a bounded interval    ← the usual choice
SESSION       your own writes are visible to you
EVENTUALLY    fastest, weakest
```

This is a real distributed-systems trade, made visible rather than hidden — which is appropriate for a system at this scale.

---

## 8. Real World Example

- **Very large-scale image and video similarity search** — the workload the architecture was built for.
- **E-commerce recommendation** over hundreds of millions of items.
- **Large enterprise RAG** across an entire document estate.
- **Research and benchmarking**, where the range of index types is valuable.

---

## 9. Milvus vs the alternatives

| | Milvus | [Qdrant](Qdrant.md) | [pgvector](pgvector.md) |
|---|---|---|---|
| **Scale ceiling** | **Billions** | High | Moderate |
| **Index variety** | **Widest** | Good | HNSW / IVFFlat |
| **Disk-based index** | **Yes (DiskANN)** | Partial | No |
| **Operational weight** | **Heavy** | Light | **None extra** |
| **Dependencies** | etcd, S3, message queue | None | None |
| **Good for prototyping** | Via Lite only | Yes | Yes |

> [!IMPORTANT]
> **Milvus is the right answer to a problem most teams do not have.** Its capabilities are real, and so is the operational burden. Choosing it for a corpus of a million vectors means running a distributed system to solve something pgvector handles on the database you already have.

---

## 10. Communication and Dependencies

- **etcd** — metadata
- **Object storage** — S3 or MinIO for the data
- **Pulsar or Kafka** — the write-ahead log
- **Kubernetes** — for the distributed deployment in practice
- **An embedding model** — Milvus stores vectors, it does not generate them

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Milvus when you genuinely have tens of millions to billions of vectors, need independent scaling of ingestion and query, or need disk-based indexing because the data will not fit in RAM.

> [!CAUTION]
> Do not use it for a first RAG system, a corpus under a few million vectors, or a team without Kubernetes operational experience. The failure mode is not that Milvus performs badly — it is that you spend your time operating a cluster instead of building the product.

---

## 12. Advantages and Disadvantages

**Advantages**
- Scales to billions of vectors
- The widest range of index types, including disk-based
- Components scale independently
- Explicit, tunable consistency
- GPU acceleration available
- Open governance under a foundation

**Disadvantages**
- **Substantial operational complexity**
- Multiple infrastructure dependencies
- Steep learning curve
- Overkill below tens of millions of vectors
- Resource-hungry even when idle
- Debugging a distributed cluster is a specialised skill

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Query** | Milliseconds, even on very large collections |
| **Scale** | Billions of vectors, with sufficient hardware |
| **Memory** | Reduced dramatically by DiskANN and quantisation |
| **Ingestion** | Very high throughput; data and index nodes scale separately |
| **Cluster baseline** | Real cost before a single vector is stored |

---

## 14. Security Considerations

- **Enable authentication and TLS** — the default deployment is open, as with its peers
- **Role-based access control** is supported; use it per collection
- **The dependencies are also attack surface** — etcd holds all metadata, and object storage holds all the data. Securing Milvus means securing three systems
- **Multi-tenancy** via partition keys or separate collections; filter on every query
- **Vectors are reconstructible** — the collection carries its source's sensitivity
- **Deletion propagation** from your primary database applies here as to any separate store

---

## 15. Mental Model

> [!NOTE]
> **pgvector is a filing cabinet in your office. Qdrant is a dedicated records room. Milvus is a warehouse with forklifts, a loading bay and a logistics team.**
>
> The warehouse is the only option when you have a million boxes. It is an absurd amount of infrastructure for two hundred.

---

## 16. Mini Architecture Diagram

```text
Application → Proxy
                ↓
   ┌────────────┼────────────┐
Query nodes  Data nodes  Index nodes
   └────────────┼────────────┘
                ↓
   etcd (metadata) · object storage (data) · message queue (WAL)
```

---

## 17. Complete Request Flow

```text
─────────── ingestion ───────────
Vectors inserted via the proxy
    ↓
Written to the message queue (durability)
    ↓
Data nodes persist segments to object storage
    ↓
Index nodes build indexes asynchronously
    ↓
Query nodes load indexed segments
    ↓
─────────── query ───────────
Search request → proxy
    ↓
Fanned out across query nodes holding relevant segments
    ↓
Each searches its portion with the configured index
    ↓
Partial results merged and ranked
    ↓
Consistency level determines whether very recent writes are visible
    ↓
Top-k returned in a few milliseconds
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Milvus scales to billions of vectors through a distributed architecture — and that architecture is the cost, which only makes sense at a scale most systems never reach.

---

## 19. Common Mistakes

- **Choosing it for a small corpus**, then operating a cluster unnecessarily
- **Underestimating the dependencies** — etcd, object storage and a message queue are all yours to run
- **Prototyping on Standalone**, then discovering Distributed behaves differently
- **Wrong index type** — FLAT on a large collection, or HNSW where memory is insufficient
- **Ignoring consistency levels**, then being surprised that a recent write is not visible
- **No authentication**
- **Securing Milvus but not etcd or the object store**

---

## 20. Open Source Technologies

- **Milvus**, **Milvus Lite**, **Zilliz Cloud**
- **DiskANN**, **FAISS**, **HNSW** — the index implementations underneath
- **etcd**, **MinIO**, **Pulsar** — the required infrastructure
- **pgvector**, **Qdrant** — the options to rule out first
- **VectorDBBench** — benchmarking across vector databases

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Count your actual vectors and compare against pgvector's comfortable range before considering Milvus.
- [ ] If you deploy it, list every dependency and who operates and backs up each one.
- [ ] Try Milvus Lite to learn the API without deploying a cluster.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
App → proxy → query/data/index nodes → etcd + object storage + message queue
```

## 2. Request Flow

```text
Input       a query vector, filters and a consistency level
    ↓
Processing  fanned out across query nodes, merged and ranked
    ↓
Output      top-k results at the requested consistency
```

## 3. Real-World Usage

**Large-scale image similarity search** is Milvus's natural home: hundreds of millions of vectors, high ingestion rates, and a genuine need to scale query and ingestion independently. Below that scale, its architecture is cost without benefit.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A distributed vector database for very large-scale similarity search |
| **Why does it exist?** | Because billions of vectors do not fit on one machine |
| **Where does it belong?** | On Kubernetes, beside object storage and etcd |
| **When should I use it?** | Tens of millions of vectors and up — not as a starting point |

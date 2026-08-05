# Embeddings

> **In one line —** turning text, images or anything else into a list of numbers where *distance means similarity*, which is what makes search by meaning possible.

| | |
|---|---|
| **Category** | Representation Technique |
| **Architectural Layer** | Data / Model |
| **Related notes** | [Vector Databases](Vector%20Databases.md) · [RAG](RAG.md) · [NLP](NLP.md) · [Transformers](Transformers.md) · [pgvector](pgvector.md) |

---

## 1. Short Definition

*What is it?*

An embedding is a fixed-length vector of numbers representing a piece of content, produced by a model such that **semantically similar content produces nearby vectors**.

---

## 2. Purpose

*What is its main purpose?*

To convert meaning into geometry. Once meaning is a position in space, "find similar things" becomes "find nearby points" — a problem computers solve very well.

---

## 3. Problem

*What engineering problem does it solve?*

```text
KEYWORD SEARCH
query "car"  →  no match for "automobile", "vehicle", "sedan"
    ↓
exact string matching cannot see meaning

EMBEDDINGS
"car"        → [0.21, -0.45, 0.88, ...]
"automobile" → [0.19, -0.43, 0.91, ...]   ← nearly the same position
    ↓
similarity is a distance calculation
```

---

## 4. Architecture Position

```text
Content (text, image, audio)
    ↓
Embedding model
    ↓
Vector  [0.21, -0.45, 0.88, ... ]  (typically hundreds to a few thousand numbers)
    ↓
Vector database / index
    ↓
Nearest-neighbour search
```

---

## 5. What the numbers mean

Nothing individually. No dimension corresponds to a human concept.

```text
The meaning is entirely RELATIONAL:
    vectors that are close have similar meaning
    the direction between vectors can carry relationships
    ↓
The classic demonstration:
    king − man + woman ≈ queen
```

> [!IMPORTANT]
> This is why embeddings from **different models are not comparable**. Each model builds its own coordinate system. Mixing vectors from two models produces meaningless distances — and it is a silent failure, because the arithmetic still works.

---

## 6. Similarity measures

```text
COSINE SIMILARITY    angle between vectors, ignores magnitude
                     → the default for text
DOT PRODUCT          angle and magnitude
                     → when vector length carries meaning
EUCLIDEAN DISTANCE   straight-line distance
                     → images, and some embedding models
```

> [!TIP]
> **Use the metric the model was trained with.** Model documentation states this, and using the wrong one degrades results quietly rather than obviously. For most text models, that is cosine similarity on normalised vectors.

---

## 7. Chunking — the decision that determines quality

For text longer than a model's input limit, you must split it. This choice affects retrieval quality more than the model does.

```text
TOO SMALL   "the total was €500"
            → no context; which invoice? which customer?

TOO LARGE   an entire 40-page document as one vector
            → the meaning is averaged away; everything looks vaguely similar

GOOD        a paragraph or a section, with some overlap between chunks
            → self-contained meaning, retrievable, specific
```

> [!IMPORTANT]
> **Chunking is where most retrieval systems are won or lost.** Teams commonly try three embedding models before revisiting the chunking, when the chunking was the problem. Overlap between chunks (a sentence or two) prevents meaning being severed at a boundary.

---

## 8. What embeddings do and do not capture

```text
✓ Topic and subject matter
✓ Semantic similarity
✓ Paraphrase and synonym
✓ Language-crossing meaning, with multilingual models

✗ NEGATION           "safe for children" and "not safe for children"
                     embed very closely — a genuine and dangerous limitation
✗ Exact identifiers  order numbers, SKUs, names
✗ Numerical ranges   "under €500" is not a geometric concept
✗ Recency or authority
```

> [!CAUTION]
> **Negation is the failure that catches people out.** Two sentences that mean opposite things frequently produce near-identical embeddings, because they share almost all their vocabulary and topic. Semantic search cannot be relied upon to distinguish them.

---

## 9. Hybrid search

```text
SEMANTIC   good for descriptive queries: "how do I cancel my subscription"
KEYWORD    good for exact terms: "INV-2026-0847", "GDPR Article 17"
    ↓
COMBINE THEM
```

> [!TIP]
> **Hybrid search beats either approach alone in almost every production system.** Run both, then merge results with reciprocal rank fusion. The most common complaint about semantic search — "it cannot find the document when I paste the exact title" — is solved entirely by adding keyword search back.

---

## 10. Real World Example

- **Semantic search** over documentation, support articles and internal wikis.
- **[RAG](RAG.md)** — retrieval is embedding-based nearest-neighbour search.
- **Recommendations** — "similar products", "related articles".
- **Deduplication and clustering** — finding near-identical records.
- **Multimodal search** — text queries against images, using a model that embeds both into one space.

---

## 11. Communication and Dependencies

- **An embedding model** — local (sentence-transformers) or hosted
- **A [vector database](Vector%20Databases.md)** or index
- **Consistent preprocessing** between indexing and querying
- **A re-embedding plan** — changing model means rebuilding the entire index

---

## 12. The model-change cost

> [!CAUTION]
> **Changing embedding model invalidates every stored vector.** There is no migration — the coordinate systems are unrelated. You must re-embed the whole corpus, which for a large one is a real project with real cost.

Plan for this: store the source text alongside the vectors, record which model produced them, and treat the model as a versioned dependency.

---

## 13. When To Use / When NOT To Use

> [!TIP]
> Use embeddings when meaning matters more than wording: search over prose, recommendations, clustering, and retrieval for RAG.

> [!CAUTION]
> - **Not for exact lookups** — an order number belongs in a database index
> - **Not for filtering by attributes** — "orders over €500 in July" is SQL, not similarity
> - **Not where negation matters** without additional verification
> - **Not for small corpora** — with a few hundred documents, keyword search and a good ranking may be entirely sufficient

---

## 14. Advantages and Disadvantages

**Advantages**
- Finds meaning rather than spelling
- Works across languages with multilingual models
- Same technique applies to text, images and audio
- Fast to search, once indexed
- Cheap to compute and cacheable

**Disadvantages**
- **Fails on negation**
- Poor at exact identifiers
- Model change means full re-indexing
- Chunking quality dominates results
- Storage cost — vectors are large relative to the text
- Not interpretable; you cannot explain why two things matched

---

## 15. Performance Impact

| Aspect | Impact |
|---|---|
| **Generation** | 10–50 ms per chunk; batch for throughput |
| **Storage** | A 1,024-dimension float32 vector is ~4 KB — for a million chunks, ~4 GB |
| **Search** | Sub-millisecond with an approximate index |
| **Caching** | Unchanged text never needs re-embedding |

> [!TIP]
> **Cache aggressively and hash your chunks.** Re-embedding an unchanged corpus is one of the most common avoidable costs in these systems. Quantising vectors to 8-bit reduces storage several-fold with modest accuracy loss.

---

## 16. Security Considerations

> [!CAUTION]
> **Embeddings are not anonymised data.** Research has repeatedly shown that original text can be substantially reconstructed from its embedding. A vector store containing embeddings of medical notes or private messages holds personal data, and treating it as "just numbers" is a mistake with legal consequences.

- **Apply the same access controls** to a vector store as to the source documents
- **Filter by permission at query time** — a user must not retrieve chunks from documents they cannot read. This is the most common access-control failure in RAG systems
- **Deletion must remove vectors too** — a deletion request is not satisfied by removing the source row
- **Embedding via a hosted API sends your content to a third party**
- **Poisoned documents** in the corpus influence retrieval for every user

---

## 17. Mental Model

> [!NOTE]
> **An embedding is a map coordinate for meaning.**
>
> Every idea gets a position. Related ideas sit near each other; unrelated ones sit far apart. You cannot read the coordinates and understand anything — but you can measure distances, and that is enough to answer "what is near this?"

---

## 18. Mini Architecture Diagram

```text
Documents
    ↓
CHUNK  (paragraph-sized, with overlap)     ← the decisive step
    ↓
Embedding model → vectors
    ↓
Vector index + metadata (source, permissions, model version)
    ↓
Query → embedded with the SAME model
    ↓
Nearest neighbours, filtered by permission
    ↓
Merged with keyword results (hybrid)
```

---

## 19. Complete Request Flow

```text
─────────── indexing ───────────
Document ingested
    ↓
Split into ~500-token chunks with 50-token overlap
    ↓
Each chunk hashed — unchanged chunks skipped
    ↓
Embedded in batches
    ↓
Stored with: source id, permissions, model version, original text
    ↓
─────────── querying ───────────
"how do I cancel my subscription"
    ↓
Embedded with the SAME model
    ↓
Vector search → top 20 candidates
    ↓
FILTERED BY THE USER'S PERMISSIONS        ← before anything is returned
    ↓
Merged with keyword search results
    ↓
Reranked → top 3 passed to the LLM
    ↓
─────────── model upgrade ───────────
New embedding model chosen
    ↓
Entire corpus re-embedded into a NEW index
    ↓
Traffic switched only once the new index is complete
```

---

## 20. Key Takeaway

> [!IMPORTANT]
> Embeddings turn meaning into distance — chunking determines quality more than the model does, hybrid search beats semantic alone, and negation is a genuine blind spot.

---

## 21. Common Mistakes

- **Mixing vectors from different models**
- **Chunks too large or too small**, with no overlap
- **Semantic search alone**, then wondering why exact titles do not match
- **No permission filtering** on retrieval — a real data leak
- **Re-embedding unchanged content** repeatedly
- **Assuming embeddings are anonymised**
- **Expecting negation to work**
- **No record of which model produced which vectors**

---

## 22. Open Source Technologies

- **sentence-transformers** — local text embeddings
- **CLIP** — text and images in one space
- **Qdrant**, **Weaviate**, **Milvus**, **pgvector** — storage and search
- **BM25 / Elasticsearch** — the keyword half of hybrid search
- **rerankers** (cross-encoders) — improve the final ordering considerably

---

## 23. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 24. Workbook Exercise

- [ ] Embed two sentences that mean opposite things and measure their similarity.
- [ ] Try three chunk sizes on the same corpus and compare retrieval quality.
- [ ] Check whether your retrieval filters by user permissions before returning results.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Content → chunking → embedding model → vector index → similarity search
```

## 2. Request Flow

```text
Input       a chunk of content, or a query
    ↓
Processing  converted to a vector in a learned semantic space
    ↓
Output      nearest neighbours by distance, filtered and reranked
```

## 3. Real-World Usage

**Hybrid search** is what production systems converge on: semantic search for descriptive queries, keyword search for identifiers, results merged. Pure semantic search demos well and disappoints in production for exactly the reasons in section 8.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A numeric vector where distance represents semantic similarity |
| **Why does it exist?** | Because exact matching cannot find meaning |
| **Where does it belong?** | Between raw content and a vector search index |
| **When should I use it?** | Meaning-based search and retrieval — combined with keyword search |

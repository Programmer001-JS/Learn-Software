# RAG

> **In one line —** retrieve the relevant documents first, then ask the model to answer using only those — which turns a system that invents facts into one that quotes yours.

| | |
|---|---|
| **Full name** | Retrieval-Augmented Generation |
| **Category** | Architecture Pattern |
| **Architectural Layer** | Application |
| **Related notes** | [Embeddings](Embeddings.md) · [Vector Databases](Vector%20Databases.md) · [LLM Fundamentals](LLM%20Fundamentals.md) · [pgvector](pgvector.md) · [AI Agents](AI%20Agents.md) |

---

## 1. Short Definition

*What is it?*

RAG is a pattern where, before answering, the system retrieves relevant passages from your own data and includes them in the prompt. The model answers **from the provided text** rather than from memory.

---

## 2. Purpose

*What is its main purpose?*

To make a language model answer about **your** data — accurately, with citations, and without retraining anything.

---

## 3. Problem

*What engineering problem does it solve?*

```text
"What is our refund policy for enterprise customers?"
    ↓
The model has never seen your policy
    ↓
It produces a plausible, well-written, entirely invented answer
    ↓
─────────── with RAG ───────────
Retrieve the actual policy sections
    ↓
"Answer using only the passages below. Cite them."
    ↓
A correct answer, with sources the user can check
```

---

## 4. Architecture Position

```text
─── INDEXING (offline) ───
Documents → chunk → embed → vector store + metadata

─── QUERYING (per request) ───
Question → embed → retrieve → filter by permission → rerank
    ↓
Prompt: instructions + passages + question
    ↓
LLM → answer + citations
    ↓
Verify citations → return
```

---

## 5. Where quality actually comes from

> [!IMPORTANT]
> Teams consistently expect the model to be the deciding factor. It is not. In practice the ranking is roughly:

```text
CHUNKING QUALITY      ████████████
RETRIEVAL QUALITY     ██████████
RERANKING             ██████
PROMPT                ████
MODEL CHOICE          ██
```

**If the retrieval does not surface the right passage, no model can answer correctly.** Most disappointing RAG systems are retrieval failures being blamed on the model.

---

## 6. Chunking

```text
TOO SMALL     "the limit is 30 days"
              → 30 days for what? Which policy? Which customer type?

TOO LARGE     an entire 40-page handbook as one chunk
              → the embedding averages everything; nothing is specific

GOOD          a section or a few paragraphs, with overlap
              → self-contained, retrievable, specific
```

```text
Practical starting point:
    ~500 tokens per chunk, ~50 tokens of overlap
    split on semantic boundaries — headings, sections — not character counts
    include the document title and section heading IN the chunk text
```

> [!TIP]
> **Prepending the document title and heading to each chunk is one of the cheapest quality improvements available.** A chunk reading "…must be requested within 30 days" becomes "Refund Policy › Enterprise Customers: …must be requested within 30 days" — vastly more retrievable, and the model has the context to answer correctly.

---

## 7. Retrieval

```text
Query
    ↓
HYBRID SEARCH   semantic (embeddings) + keyword (BM25)
    ↓
Top 20–50 candidates
    ↓
PERMISSION FILTER  ← must happen inside the search
    ↓
RERANK with a cross-encoder → top 3–5
    ↓
Into the prompt
```

> [!TIP]
> **Reranking is the highest-value addition after hybrid search.** A cross-encoder scores each candidate against the query directly, rather than comparing pre-computed vectors. It is slower per item, which is why you rerank 20 candidates rather than the whole corpus — and it consistently improves the final ordering.

---

## 8. The prompt

```text
"Answer the question using ONLY the passages below.
 Quote the relevant text and cite the source id.
 If the passages do not contain the answer, say so.
 Do not use any other knowledge."

[passages with ids]

Question: ...
```

> [!IMPORTANT]
> **"If the answer is not in the passages, say so" is the single most important instruction**, and it must be paired with mechanical verification — because the model will sometimes answer anyway.

---

## 9. Verify the citations

```text
Model returns: answer + citations
    ↓
CHECK IN CODE: does each quoted string actually appear in the retrieved passages?
    ↓ NO  → reject the answer; return "I could not find this" with links
    ↓ YES → return the answer with sources
```

> [!TIP]
> This mechanical check is cheap, deterministic, and catches the failure that matters most. It is far more reliable than any amount of prompt wording, and it is skipped in most implementations.

---

## 10. RAG vs fine-tuning

```text
RAG                                   FINE-TUNING
teaches KNOWLEDGE                     teaches STYLE, FORMAT, BEHAVIOUR
updates instantly — reindex a doc     requires retraining
citations available                   no citations
permission filtering possible         no per-user access control
cheaper                               expensive, and goes stale
```

> [!CAUTION]
> **"Fine-tune the model on our documentation" is almost always the wrong answer.** Fine-tuning does not reliably teach facts, produces no citations, cannot enforce permissions, and is stale the moment a document changes. If the goal is "answer from our data", the answer is RAG.

---

## 11. Real World Example

- **Internal documentation assistants** — the most common deployment.
- **Customer support**, drafting answers grounded in the knowledge base.
- **Legal and contract question answering**, with human verification.
- **Product search and recommendation** with natural-language queries.

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use RAG when answers must come from a corpus you control, must be current, must cite sources, or must respect per-user permissions.

> [!CAUTION]
> - **Not for aggregate questions** — "how many contracts mention indemnity?" is a database query, not a retrieval problem. RAG retrieves a handful of passages; it cannot count across a corpus
> - **Not for a small corpus** — with twenty documents, put them all in the context and skip the pipeline
> - **Not where the answer requires reasoning across many documents** at once
> - **Not as a substitute for search** when the user actually wants a list of documents

---

## 13. Advantages and Disadvantages

**Advantages**
- Answers grounded in your data, with citations
- Updates immediately when a document changes
- Permission filtering per user is possible
- No training required
- Far cheaper than fine-tuning
- The user can verify the source

**Disadvantages**
- Quality is dominated by retrieval, which is harder than it looks
- A pipeline with several failure points
- Latency: embed → retrieve → rerank → generate
- Chunking is fiddly and corpus-specific
- Cannot answer aggregate or cross-document questions
- Evaluation requires deliberate effort

---

## 14. Evaluation

> [!IMPORTANT]
> **Without an evaluation set, you cannot tell whether a change helped.** RAG systems are full of tempting adjustments — chunk size, retrieval count, prompt wording — and each one improves some queries and degrades others. "It seems better" is not a measurement.

```text
Build 50–100 question/answer pairs with known source passages
    ↓
Measure separately:
    RETRIEVAL   was the correct passage in the top-k?      ← measure this FIRST
    ANSWER      is the generated answer correct?
    GROUNDING   is every claim supported by a passage?
```

Measuring retrieval separately is essential, because it tells you whether a wrong answer is a retrieval problem or a generation problem — and they have completely different fixes.

---

## 15. Performance Impact

| Stage | Latency |
|---|---|
| **Embed the query** | 10–50 ms |
| **Vector search** | 1–10 ms |
| **Rerank** | 50–200 ms |
| **Generation** | Hundreds of ms to seconds — the dominant cost |
| **Total** | Typically 1–3 seconds |

> [!TIP]
> **Stream the response.** Total time is unchanged, but the user sees an answer forming immediately instead of waiting several seconds for a complete one. This single change makes a RAG system feel fast.

---

## 16. Security Considerations

> [!CAUTION]
> **The most common serious RAG defect is retrieving documents the user is not permitted to see.** The vector store contains everything; if the query does not filter by permission, a well-phrased question can surface another team's salary spreadsheet — and the answer will helpfully summarise it.

- **Filter by permission inside the retrieval query**, never afterwards
- **Deletion must remove vectors**, not just source rows
- **Indirect prompt injection** — a poisoned document in the corpus carries instructions that reach the model on every retrieval. This is the most serious RAG-specific attack, and there is no complete defence: treat retrieved content as untrusted, restrict what the model can do, and never let it trigger actions unsupervised
- **Embeddings are reconstructible** — the vector store carries the sensitivity of the source
- **Log what was retrieved for each answer** — you will need it for both debugging and audit

---

## 17. Mental Model

> [!NOTE]
> **Without RAG, you are asking someone to answer from memory. With RAG, you hand them the file first and ask them to quote from it.**
>
> The same person gives a far more reliable answer with the file in front of them — and you can check their quotes against the pages.

---

## 18. Mini Architecture Diagram

```text
─── indexing ───
Docs → chunk (with headings) → embed → vector store + metadata + ACL

─── query ───
Question → embed
    ↓
Hybrid search + PERMISSION FILTER
    ↓
Rerank → top 3–5
    ↓
Prompt: instructions + passages + question
    ↓
LLM (streamed)
    ↓
VERIFY citations against the passages
    ↓
Answer + sources, or an honest "not found"
```

---

## 19. Complete Request Flow

```text
"What is our refund policy for enterprise customers?"
    ↓
Query embedded
    ↓
Hybrid search: semantic + BM25
    ↓
FILTER: only documents this user may read
    ↓
20 candidates → cross-encoder rerank → top 3
    ↓
Prompt assembled with the passages and their ids
    ↓
LLM generates, streamed to the user
    ↓
Citations extracted and CHECKED against the passages
    ↓ one quote not found
Answer rejected → "I could not find this — here are the closest documents"
    ↓
Retrieved passages and the outcome logged for evaluation
    ↓
─────────── a document is updated ───────────
Re-chunked and re-embedded; old vectors removed
    ↓
The next question reflects the change immediately — no retraining
```

---

## 20. Key Takeaway

> [!IMPORTANT]
> RAG quality is retrieval quality — fix chunking and reranking before changing models, filter permissions inside the search, and verify citations mechanically.

---

## 21. Common Mistakes

- **Blaming the model** when retrieval is the problem
- **Fixed-size chunking** that severs meaning mid-sentence
- **No document title or heading** in the chunk text
- **Semantic search only**, with no keyword component
- **No reranking**
- **No permission filtering** — the most serious defect
- **No evaluation set**, so changes are guesswork
- **Fine-tuning instead of RAG** to teach facts
- **Not verifying citations**

---

## 22. Open Source Technologies

- **LangChain**, **LlamaIndex**, **Haystack** — orchestration
- **pgvector**, **Qdrant**, **Weaviate** — retrieval
- **sentence-transformers**, **BM25** — the two halves of hybrid search
- **Cross-encoder rerankers** — the largest quality gain after hybrid
- **Ragas**, **DeepEval**, **promptfoo** — evaluation
- **Langfuse**, **Phoenix** — tracing what was retrieved and why

---

## 23. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 24. Workbook Exercise

- [ ] Build 30 question/answer pairs and measure retrieval accuracy separately from answer accuracy.
- [ ] Add the document title and heading to each chunk and re-measure.
- [ ] Check whether a user can retrieve a passage from a document they cannot open directly.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Docs → chunks → vectors → retrieval + filter → rerank → prompt → LLM → verified answer
```

## 2. Request Flow

```text
Input       a question and a user identity
    ↓
Processing  retrieve permitted passages, rerank, ground the model in them
    ↓
Output      an answer with verifiable citations, or an honest failure
```

## 3. Real-World Usage

**Internal documentation assistants** are RAG's most successful deployment, and the pattern that makes them work is consistent: hybrid retrieval, reranking, permission filtering in the query, and mechanical citation verification before the answer is shown.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Retrieving relevant passages and grounding the model's answer in them |
| **Why does it exist?** | Because models invent facts they were never given |
| **Where does it belong?** | Between your document corpus and the model |
| **When should I use it?** | Answers must come from your data, be current, and cite sources |

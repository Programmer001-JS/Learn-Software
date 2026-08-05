# NLP

> **In one line —** making software work with human language, a field that was rebuilt almost entirely once transformers arrived.

| | |
|---|---|
| **Full name** | Natural Language Processing |
| **Category** | Discipline |
| **Architectural Layer** | Application |
| **Related notes** | [LLM Fundamentals](LLM%20Fundamentals.md) · [Transformers](Transformers.md) · [Embeddings](Embeddings.md) · [Machine Learning](Machine%20Learning.md) · [RAG](RAG.md) |

---

## 1. Short Definition

*What is it?*

NLP is the field concerned with getting software to process, understand and generate human language — classification, extraction, translation, summarisation, search and question answering.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Language is ambiguous, contextual and irregular
    ↓
"The bank was steep"  vs  "The bank was closed"
"I saw her duck"      — two entirely different sentences
    ↓
Rules cannot cover it; the space of valid sentences is unbounded
```

---

## 3. The historical arc

```text
1960s–1990s   RULES              hand-written grammars — brittle, enormous effort
2000s–2010s   STATISTICAL ML     bag of words, TF-IDF, naive Bayes, SVM
2013          WORD EMBEDDINGS    word2vec — words as vectors with meaning
2018          TRANSFORMERS       BERT and successors — context-aware
2020s         LARGE LANGUAGE MODELS  one model, many tasks, no task-specific training
```

> [!IMPORTANT]
> **The practical consequence of the last step is large.** Tasks that previously required a labelled dataset and a trained classifier per task — sentiment, intent, extraction, summarisation — can now often be done with a prompt and no training data at all. That does not make the earlier approaches obsolete, but it changes the default starting point.

---

## 4. The core tasks

| Task | Question |
|---|---|
| **Classification** | Is this review positive? Which department should this ticket go to? |
| **Named entity recognition** | Which words are people, places, dates, amounts? |
| **Sentiment analysis** | What is the emotional tone? |
| **Summarisation** | What are the key points? |
| **Translation** | Say this in another language |
| **Question answering** | Answer this from this document |
| **Semantic search** | Find documents about this meaning, not this wording |

---

## 5. Architecture Position

```text
Text input
    ↓
Preprocessing  (normalise, sometimes tokenise)
    ↓
┌─────────────────┬─────────────────┬──────────────────┐
Classical ML      Fine-tuned model   Language model
TF-IDF + SVM      BERT-family        prompt-based
    ↓                 ↓                   ↓
fast, cheap       fast, accurate     flexible, no training
needs labels      needs labels       costs per call
```

---

## 6. Choosing an approach

> [!TIP]
> This is the decision that matters most, and the cheapest option is far more often correct than current fashion suggests.

```text
Is it a simple, high-volume, well-defined classification?
    ↓ YES → classical ML or a small fine-tuned model
            cheap, fast, runs locally, deterministic cost

Do you have thousands of labelled examples and need low latency?
    ↓ YES → fine-tune a small transformer
            excellent accuracy, milliseconds, no per-call cost

Is the task varied, low-volume, or hard to specify?
    ↓ YES → a language model with a prompt
            no training data, immediate, costs per call
```

> [!CAUTION]
> **Calling a large language model to classify a support ticket into five categories is usually the wrong engineering choice** at volume: slower, more expensive per item, non-deterministic, and dependent on an external service. A fine-tuned small model does it in milliseconds for nothing. Use the large model to *generate the labelled data*, then train the small one.

---

## 7. Tokenisation

```text
"unbelievable"  →  ["un", "believ", "able"]
```

Models do not see words; they see **tokens** — subword units.

```text
Consequences:
    a "1,000 word" limit is not a "1,000 token" limit
    non-English text often uses far more tokens per word
    code, URLs and identifiers tokenise inefficiently
    cost and context limits are measured in tokens, not characters
```

---

## 8. Embeddings changed search

```text
KEYWORD SEARCH        "car" does not match "automobile"
                      exact terms only

SEMANTIC SEARCH       both become nearby vectors
                      matches meaning, not spelling
```

This is the foundation of [RAG](RAG.md) and modern search. See [Embeddings](Embeddings.md).

> [!TIP]
> **Hybrid search — keyword plus semantic — beats either alone** in most real systems. Keyword search still wins for exact identifiers, product codes and names; semantic search wins for descriptive queries.

---

## 9. Real World Example

- **Support ticket routing and triage** — high volume, well-defined, a classic fine-tuning case.
- **Content moderation** — classification at scale, with human review of uncertain cases.
- **Semantic search** over internal documentation.
- **Contract and document analysis** — entity extraction with human verification.
- **Translation** — the most complete transition from rules to neural methods.

---

## 10. Language coverage

> [!CAUTION]
> **English is dramatically better served than everything else.** Models, datasets, tokenisers and evaluation benchmarks are overwhelmingly English-centric. Accuracy on other languages is lower, tokenisation is less efficient (so costs are higher), and for smaller languages the available options narrow sharply.
>
> If your users write in a language other than English, evaluate on **your actual data**, not on published benchmarks.

---

## 11. Communication and Dependencies

- **A model** — local (spaCy, a fine-tuned transformer) or an API
- **A vector database**, for semantic search — see [Vector Databases](Vector%20Databases.md)
- **Labelled data**, for the fine-tuning path
- **Evaluation data** — without it you cannot tell whether a change helped

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use NLP where text volume exceeds what people can read: triage, search, extraction, summarisation of large corpora.

> [!CAUTION]
> - **Not where a regex or a lookup would do** — extracting a well-formatted order number is not an NLP problem
> - **Not for decisions requiring guaranteed correctness** without human review
> - **Not without an evaluation set** — "it looks good in a few tests" is how NLP projects fail quietly

---

## 13. Advantages and Disadvantages

**Advantages**
- Processes volumes of text no team could read
- Semantic search finds meaning rather than spelling
- Modern models handle many tasks without task-specific training
- Strong open-source ecosystem

**Disadvantages**
- Language is ambiguous; errors are inevitable
- Bias in training data becomes bias in outputs
- Non-English performance is materially weaker
- Evaluation is hard — "good summary" resists measurement
- Costs scale with volume when using hosted models

---

## 14. Performance Impact

| Approach | Latency | Cost |
|---|---|---|
| **Classical (TF-IDF + SVM)** | Sub-millisecond | Negligible |
| **Small fine-tuned transformer** | 5–50 ms | Own hardware |
| **Large hosted model** | Hundreds of ms to seconds | Per call |
| **Embedding generation** | 10–50 ms | Cheap, cacheable |

> [!TIP]
> **Cache embeddings.** Text that has not changed does not need re-embedding, and re-embedding a corpus is one of the most common avoidable costs in a RAG system.

---

## 15. Security Considerations

> [!CAUTION]
> **Text sent to a hosted model leaves your infrastructure.** For support tickets, contracts, medical notes or anything personal, that is a data processing decision requiring a lawful basis and a review of the provider's retention terms — not a technical detail.

- **Prompt injection** — user text that instructs the model is a genuine and unsolved vulnerability class; see [LLM Fundamentals](LLM%20Fundamentals.md)
- **Personal data in text** — names, addresses and identifiers appear constantly; consider redaction before processing
- **Models can memorise training data** and reproduce it
- **Bias** in classification of language has produced documented discriminatory outcomes, particularly in moderation and hiring
- **Never let model output reach a shell, a query or an eval** without validation

---

## 16. Mental Model

> [!NOTE]
> **Classical NLP counted words; modern NLP reads sentences.**
>
> Counting works surprisingly well for "is this spam?" — spam has distinctive vocabulary. It fails completely for "is this sarcastic?", because sarcasm is entirely about context and the words themselves may be positive.

---

## 17. Mini Architecture Diagram

```text
Text
    ↓
Tokenise
    ↓
┌── classify ──┬── extract ──┬── embed ──┬── generate ──┐
   small model    NER model    embedding    language model
    ↓              ↓             ↓             ↓
 routing        structured   vector DB      summary
                  fields     (search)       or answer
    ↓
Confidence check → human review where uncertain
```

---

## 18. Complete Request Flow

Support ticket triage:

```text
Ticket submitted
    ↓
Personal data redacted where possible
    ↓
Classified by a FINE-TUNED SMALL MODEL — 15 ms, runs locally
    ↓
Category: "billing", confidence 0.93
    ↓
Entities extracted: order number, amount, date
    ↓
Embedded and matched against similar past tickets
    ↓
Confidence high → routed automatically to the billing queue
Confidence low  → routed to general triage for a human
    ↓
Agent's correction is recorded
    ↓
Corrections accumulate → the model is retrained periodically
```

> [!TIP]
> The small local model is the right choice here precisely because the volume is high and the task is narrow. A large hosted model was likely used **once**, to label the initial training set.

---

## 19. Key Takeaway

> [!IMPORTANT]
> Choose the smallest approach that solves the task — a fine-tuned small model beats a large hosted one on latency, cost and determinism for high-volume, well-defined work.

---

## 20. Common Mistakes

- **Using a large model for a simple, high-volume classification**
- **No evaluation set**, so nobody can tell whether changes help
- **Assuming English-level accuracy** on other languages
- **Keyword search alone** where semantic search is needed, or the reverse
- **Not caching embeddings**
- **Sending sensitive text to a third party** without a lawful basis
- **Treating model output as trusted input** downstream

---

## 21. Open Source Technologies

- **spaCy** — fast, production-oriented classical and neural NLP
- **Hugging Face Transformers** — pre-trained models and fine-tuning
- **sentence-transformers** — embeddings
- **NLTK** — academic and educational
- **scikit-learn** — TF-IDF and classical classifiers
- **Argilla**, **Label Studio** — annotation and evaluation

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Take one text task and estimate its cost with a hosted model versus a fine-tuned small model at your volume.
- [ ] Build an evaluation set of 100 labelled examples before changing anything.
- [ ] Test your NLP pipeline on non-English text and compare the accuracy.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Text → tokenise → classify / extract / embed / generate → validate → act
```

## 2. Request Flow

```text
Input       natural language text
    ↓
Processing  tokenised and passed to the smallest sufficient model
    ↓
Output      a label, structured fields, a vector, or generated text
```

## 3. Real-World Usage

**Support ticket triage** demonstrates the pattern that works: a large model labels a training set once, a small fine-tuned model handles the ongoing volume in milliseconds, and human corrections feed the next retraining cycle.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Processing and generating human language with software |
| **Why does it exist?** | Because language is ambiguous and rules cannot cover it |
| **Where does it belong?** | Between text input and a structured decision |
| **When should I use it?** | When text volume exceeds what people can read — with the smallest model that works |

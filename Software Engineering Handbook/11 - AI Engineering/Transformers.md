# Transformers

> **In one line —** the architecture that replaced sequential processing with attention, so every token can look at every other token at once — which is what made training on the entire internet possible.

| | |
|---|---|
| **Category** | Neural Network Architecture |
| **Architectural Layer** | Model |
| **Introduced** | *Attention Is All You Need*, 2017 |
| **Related notes** | [Deep Learning](Deep%20Learning.md) · [LLM Fundamentals](LLM%20Fundamentals.md) · [Embeddings](Embeddings.md) · [GPU Computing](GPU%20Computing.md) |

---

## 1. Short Definition

*What is it?*

The transformer is a neural network architecture built around **self-attention**: a mechanism that lets every position in a sequence directly consider every other position, in one parallel operation.

---

## 2. Problem

*What engineering problem does it solve?*

```text
RNN / LSTM (before)                   TRANSFORMER
process tokens ONE AT A TIME          process all tokens AT ONCE
word 500 must wait for word 499           ↓
    ↓                                 fully parallel → GPUs are fully used
cannot parallelise training               ↓
long-range dependencies fade          any token can attend to any other directly
```

> [!IMPORTANT]
> **Parallelisation is the real breakthrough, not accuracy.** RNNs could not use a GPU efficiently because each step depended on the previous one. Transformers turned language modelling into large matrix multiplications — exactly what GPUs do — which made training on internet-scale data economically possible.

---

## 3. Self-attention

For every token, the model asks: *which other tokens matter for understanding this one?*

```text
"The animal didn't cross the street because IT was too tired"
                                          ↑
                    attention resolves "it" → "animal"

"The animal didn't cross the street because IT was too wide"
                                          ↑
                    attention resolves "it" → "street"
```

The same word, resolved differently by context. Nobody wrote a rule for this — attention weights are learned.

---

## 4. How attention works

```text
Each token produces three vectors:
    QUERY   what am I looking for?
    KEY     what do I offer?
    VALUE   what do I contribute?
    ↓
Score every query against every key
    ↓
Softmax → attention weights
    ↓
Weighted sum of values → the token's new representation
```

```text
MULTI-HEAD ATTENTION
    several attention mechanisms in parallel
    one head may track syntax, another coreference, another topic
    → richer representation than one attention pattern could give
```

---

## 5. Positional encoding

Attention has no inherent notion of order — it sees a set, not a sequence.

```text
"dog bites man"  and  "man bites dog"
    ↓ without positional information
identical to the attention mechanism
```

So position is **added to the token representations** explicitly. It is a bolted-on solution to a limitation of the core mechanism, and improving it is an active area of work.

---

## 6. The quadratic cost

> [!CAUTION]
> **Every token attends to every other token, so attention cost scales with the square of sequence length.**

```text
   1,000 tokens →   1,000,000 attention scores
  10,000 tokens → 100,000,000
 100,000 tokens →  10,000,000,000
```

This is why long context windows are expensive rather than merely large, and why an entire research field exists around making attention cheaper — FlashAttention, sliding windows, sparse and linear attention.

---

## 7. The three shapes

```text
ENCODER-ONLY     BERT-family
                 sees the whole input at once, bidirectional
                 → classification, embeddings, extraction

DECODER-ONLY     GPT-family, most current LLMs
                 predicts the next token, sees only what came before
                 → generation

ENCODER-DECODER  T5, translation models
                 encode input, decode output
                 → translation, summarisation
```

> [!TIP]
> This distinction is practical. For **classification and embeddings**, an encoder model is smaller, faster and better suited than a generative one. Using a large decoder model to produce embeddings is a common and unnecessary expense.

---

## 8. Why it took over everything

The architecture was designed for translation and turned out to be general:

```text
Text        LLMs, translation, classification
Images      Vision Transformers — patches treated as tokens
Audio       speech recognition and generation
Video       frames as sequences
Proteins    amino acid sequences
Code        tokens like any other
```

> [!IMPORTANT]
> Anything expressible as a sequence with relationships between elements fits. That generality is why one architecture displaced several specialised ones simultaneously.

---

## 9. Real World Example

- **Every current large language model** is a decoder-only transformer.
- **BERT-family encoders** power a large share of production classification and search embeddings — quietly, and far more cheaply than generative models.
- **Vision Transformers** now compete with CNNs on image tasks.
- **AlphaFold** applied attention to protein structure prediction.

---

## 10. Communication and Dependencies

- **[GPUs](GPU%20Computing.md)** — the architecture exists to exploit them
- **Enormous training data**, for pre-training
- **PyTorch / JAX** and libraries such as Hugging Face Transformers
- **FlashAttention** and similar kernels, for memory-efficient attention

---

## 11. When To Use / When NOT To Use

> [!TIP]
> You will almost never implement a transformer — you will use a pre-trained one. Understanding the architecture matters for choosing between encoder and decoder models, and for reasoning about why context length costs what it does.

> [!CAUTION]
> Do not use a transformer for tabular data, where [gradient boosting](Machine%20Learning.md) wins, or for very small datasets without a pre-trained model to fine-tune. And do not use a large decoder model where a small encoder would do the job.

---

## 12. Advantages and Disadvantages

**Advantages**
- Fully parallel training — the reason scale became possible
- Direct long-range dependencies, without degradation
- Transfers across modalities
- Extremely well-supported by tooling and hardware

**Disadvantages**
- **Quadratic attention cost** in sequence length
- Very large memory requirements
- Data-hungry — pre-training requires enormous corpora
- Position handling is an add-on rather than intrinsic
- Attention weights are only loosely interpretable, despite appearances

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Training** | Parallel across the sequence — the entire point |
| **Inference (generation)** | **Sequential** — one token at a time, so latency scales with output length |
| **Memory** | Dominated by attention and the KV cache |
| **Context length** | Quadratic in attention; the main cost driver |
| **KV caching** | Essential for generation; avoids recomputing past attention |

> [!IMPORTANT]
> Note the asymmetry: **training is parallel, generation is not.** A transformer generates one token at a time, each conditioned on all previous ones. That is why output length drives latency and why streaming exists — the total time is unchanged, but the user sees progress.

---

## 14. Security Considerations

- **Attention over untrusted content is the mechanism behind prompt injection** — the model attends to instructions embedded in retrieved documents exactly as it attends to yours, because architecturally they are the same tokens. There is no separation to enforce
- **Pre-trained weights are executable artifacts** when loaded from pickle-based formats; prefer safetensors
- **Models memorise training data**, and extraction attacks have recovered verbatim sequences
- **Long contexts increase the injection surface** — every retrieved document is more text that can carry instructions

---

## 15. Mental Model

> [!NOTE]
> **An RNN reads a sentence word by word, remembering what it can. A transformer lays the whole sentence on a table and lets every word look at every other word simultaneously.**
>
> The table approach is far faster and never forgets the beginning of the sentence. Its cost is that the table must be big enough for everything at once — and the table area grows with the square of the sentence length.

---

## 16. Mini Architecture Diagram

```text
Tokens
    ↓
Embeddings + positional encoding
    ↓
┌──── Transformer block  (×N) ────┐
│  Multi-head self-attention       │
│  Add & normalise                 │
│  Feed-forward network            │
│  Add & normalise                 │
└──────────────┬───────────────────┘
               ↓
Output head → probabilities over the vocabulary
```

---

## 17. Complete Request Flow

Generating a response:

```text
Prompt tokenised
    ↓
Each token embedded, position added
    ↓
Through N transformer blocks:
    every token attends to every other token
    representations become increasingly contextual
    ↓
Final layer → probability distribution over the vocabulary
    ↓
One token sampled and appended
    ↓
REPEAT — but with KV caching, past attention is not recomputed
    ↓
Each new token still requires a full forward pass
    ↓
Which is why output length, not input length, dominates latency
    ↓
Stop token reached → generation ends
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Attention lets every token see every other token in parallel, which made large-scale training possible — and its quadratic cost is why context length is expensive.

---

## 19. Common Mistakes

- **Using a large decoder model for embeddings** where a small encoder is better and cheaper
- **Assuming longer context is free** — it costs quadratically in attention
- **Expecting parallel generation** — output is inherently sequential
- **Reading attention weights as explanations** — they are suggestive, not definitive
- **Loading model weights from untrusted pickle files**
- **Choosing a transformer for tabular data**

---

## 20. Open Source Technologies

- **Hugging Face Transformers** — the standard library
- **FlashAttention** — memory-efficient attention kernels
- **vLLM** — high-throughput serving with efficient KV caching
- **safetensors** — safe weight serialisation
- **PyTorch**, **JAX** — the frameworks underneath
- *Attention Is All You Need* (2017) — short, readable, and worth reading directly

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Work out the attention cost difference between a 1,000-token and a 10,000-token context.
- [ ] Compare an encoder model and a decoder model for a classification task, on speed and cost.
- [ ] Measure how response latency changes with output length versus input length.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Tokens → embeddings + position → N attention blocks → output distribution
```

## 2. Request Flow

```text
Input       a token sequence
    ↓
Processing  every token attends to every other, through stacked blocks
    ↓
Output      contextual representations, or one generated token at a time
```

## 3. Real-World Usage

**Vision Transformers** applied the same architecture to images by treating patches as tokens, and now compete with convolutional networks. One architecture displacing specialised ones across several fields is unusual and is the strongest evidence of its generality.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A neural architecture based on parallel self-attention |
| **Why does it exist?** | Because sequential models could not be parallelised or scaled |
| **Where does it belong?** | As the model layer beneath essentially all modern AI |
| **When should I use it?** | Via pre-trained models — encoder for understanding, decoder for generation |

# AI Fundamentals

> **In one line —** software that learns patterns from data instead of being told the rules, which makes it powerful where rules cannot be written and unreliable where they can.

| | |
|---|---|
| **Category** | Overview note *(hub for this section)* |
| **Architectural Layer** | Application / Data |
| **Related notes** | [Machine Learning](Machine%20Learning.md) · [Deep Learning](Deep%20Learning.md) · [LLM Fundamentals](LLM%20Fundamentals.md) · [GPU Computing](GPU%20Computing.md) · [AI Pipelines](AI%20Pipelines.md) |

---

## 1. Short Definition

Artificial intelligence, in practice, means systems that derive their behaviour from **data** rather than from explicit instructions. You do not write the rules; you provide examples and the system infers a function that fits them.

---

## 2. The distinction that matters

```text
TRADITIONAL SOFTWARE           MACHINE LEARNING
rules + data → answers         data + answers → rules
    ↓                              ↓
you write: if spam_words > 5   you provide: 100,000 labelled emails
    ↓                              ↓
deterministic, explainable     statistical, approximate
fails predictably              fails unpredictably
```

> [!IMPORTANT]
> **The output is a probability, not a fact.** A classifier that is 97% accurate is wrong three times in a hundred, and you cannot know in advance which three. Every design decision downstream follows from accepting that.

---

## 3. The nested terms

```text
ARTIFICIAL INTELLIGENCE       any system exhibiting intelligent behaviour
  └─ MACHINE LEARNING         learns patterns from data
       └─ DEEP LEARNING       neural networks with many layers
            └─ TRANSFORMERS   the architecture behind modern language models
                 └─ LLMs      very large transformers trained on text
```

---

## 4. Architecture Position

```text
Application
    ↓
Model serving layer  (an API, like any other service)
    ↓
Model  (weights loaded into memory or GPU VRAM)
    ↓
GPU / CPU
```

> [!TIP]
> From a systems perspective, **a model is just a service with unusual resource requirements**: large memory footprint, GPU dependency, slow cold start, and a non-deterministic response. Everything in folders 04 through 14 still applies — it needs a timeout, a fallback, monitoring and a rate limit like anything else.

---

## 5. The three learning paradigms

```text
SUPERVISED       labelled examples → predict the label
                 spam/not spam · price prediction · image classification
                 → most business applications

UNSUPERVISED     no labels → find structure
                 clustering customers · anomaly detection · embeddings

REINFORCEMENT    actions → rewards → learn a policy
                 game playing · robotics · some model fine-tuning
```

---

## 6. Where AI genuinely fits

```text
✓ The rules exist but cannot be written down
      "is this photo a cat?"  ·  "does this review sound angry?"
✓ Patterns in high-dimensional data
      fraud detection · recommendations
✓ Natural language and images
✓ Approximate answers are acceptable

✗ Exact arithmetic                    use code
✗ Rules that CAN be written           use an if statement
✗ Decisions requiring an explanation  regulated lending, medical diagnosis
✗ Anything that must be 100% correct
```

> [!CAUTION]
> The most common engineering mistake in this area is using a model where a rule would do. "Classify these three known categories" is often a lookup table, and the lookup table is faster, free, testable and correct every time.

---

## 7. What actually determines success

```text
DATA QUALITY     ████████████  the dominant factor
FEATURES         ██████
MODEL CHOICE     ███
HYPERPARAMETERS  ██
```

> [!IMPORTANT]
> Practitioners consistently report that data work — collection, cleaning, labelling — is the majority of the effort and the majority of the improvement. A better model on poor data loses to a simple model on good data, essentially always.

---

## 8. The evaluation trap

```text
Accuracy 99%!
    ↓
The dataset is 99% negative examples
    ↓
"Always predict negative" also scores 99%
    ↓
The model has learned nothing
```

For imbalanced problems — fraud, disease, defects — use **precision**, **recall** and their trade-off:

```text
PRECISION  of the ones we flagged, how many were right?   → false positives
RECALL     of the real ones, how many did we catch?       → false negatives
```

> [!TIP]
> Which one matters is a **business decision, not a technical one**. Cancer screening favours recall; a spam filter favours precision, because a lost legitimate email costs more than a spam message getting through.

---

## 9. Real World Example

- **Recommendations, fraud detection and search ranking** are the highest-value classical applications, and predate the current wave entirely.
- **Language models** made natural-language interfaces practical for ordinary applications.
- **Computer vision** in manufacturing quality control and document processing.
- **The unglamorous majority**: forecasting, churn prediction, classification, deduplication.

---

## 10. Mental Model

> [!NOTE]
> **Traditional software is a recipe; machine learning is an apprentice.**
>
> The recipe produces the same dish every time and you can read exactly why. The apprentice has watched ten thousand dishes being made, is usually right, occasionally does something strange, and cannot fully explain their reasoning. You would not use an apprentice to measure the flour — and you would not write a recipe for judging whether the sauce tastes right.

---

## 11. Key Takeaway

> [!IMPORTANT]
> AI systems produce probabilities from patterns in data — use them where rules cannot be written, and never where an exact answer is required.

---

## 12. Common Mistakes

- **Using a model where a rule would do**
- **Judging by accuracy** on an imbalanced dataset
- **Treating outputs as facts** rather than as predictions with an error rate
- **Underestimating data work** — it is most of the project
- **No fallback** for when the model is unavailable, slow or wrong
- **Ignoring model drift** — the world changes and yesterday's model quietly degrades

---

## 13. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 14. Workbook Exercise

- [ ] Take one feature you are considering AI for, and write down whether a rule could express it.
- [ ] For one classification task, decide whether precision or recall matters more, and why in business terms.
- [ ] Write down what your application does when the model is wrong.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Data → training → model → serving API → application
                              ↑
                        monitoring for drift
```

## 2. Request Flow

```text
Input       data, or a request needing a prediction
    ↓
Processing  the model applies patterns learned from training data
    ↓
Output      a probabilistic answer, correct most of the time
```

## 3. Real-World Usage

**Fraud detection** is the archetypal fit: the rules cannot be written down, they change constantly as fraudsters adapt, and a probabilistic answer with human review of the uncertain cases is genuinely better than any rule set.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Software whose behaviour is learned from data rather than specified |
| **Why does it exist?** | Because some rules cannot be written, only demonstrated |
| **Where does it belong?** | As a service behind your application, with a fallback |
| **When should I use it?** | When rules cannot express the problem and approximation is acceptable |

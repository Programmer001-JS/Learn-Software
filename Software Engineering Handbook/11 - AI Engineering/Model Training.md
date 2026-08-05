# Model Training

> **In one line —** the process that turns data into weights, and the part of AI work where almost all the effort goes into the data rather than the training.

| | |
|---|---|
| **Category** | Practice |
| **Architectural Layer** | Data / Compute |
| **Related notes** | [Machine Learning](Machine%20Learning.md) · [Deep Learning](Deep%20Learning.md) · [GPU Computing](GPU%20Computing.md) · [Model Serving](Model%20Serving.md) · [AI Pipelines](AI%20Pipelines.md) |

---

## 1. Short Definition

*What is it?*

Training is the process of adjusting a model's parameters so that its predictions on training data improve — repeatedly, until performance on **held-out** data stops improving.

---

## 2. The three kinds of training

> [!IMPORTANT]
> These differ enormously in cost, and confusing them is the source of most unrealistic expectations.

```text
FROM SCRATCH      random weights → full dataset
                  months, enormous compute, research teams
                  → you will almost certainly never do this

FINE-TUNING       pre-trained weights → your smaller dataset
                  hours on one GPU, thousands of examples
                  → the realistic option

PARAMETER-EFFICIENT (LoRA and similar)
                  freeze the base, train a small adapter
                  → minutes to hours, far less memory, small artifacts
```

---

## 3. The training loop

```text
Batch of examples
    ↓
Forward pass → predictions
    ↓
Loss function → how wrong?
    ↓
Backward pass → gradients for every parameter
    ↓
Optimiser → adjust the weights
    ↓
Repeat, over epochs
```

---

## 4. When to stop

```text
Training loss   ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓  (keeps falling — it always will)
Validation loss ↓ ↓ ↓ ↓ ↑ ↑ ↑ ↑
                        ↑
                   STOP HERE
```

> [!IMPORTANT]
> **Training loss falling is not progress.** It falls right up to the point of memorisation. The moment validation loss turns upward, the model has begun memorising rather than learning — early stopping exists precisely to catch that.

---

## 5. Fine-tuning versus the alternatives

```text
Is the problem "the model does not know our facts"?
    ↓ YES → RAG, not fine-tuning
Is the problem "the model will not follow our format or tone"?
    ↓ YES → fine-tuning fits
Is the problem "we need a small, fast, cheap model for one narrow task"?
    ↓ YES → fine-tune a small model; this is where it shines most
```

> [!CAUTION]
> **Fine-tuning does not reliably teach facts.** The model may reproduce your training examples and will still invent things outside them, with no citations and no way to update without retraining. For knowledge, [RAG](RAG.md) is the correct pattern — this is the most common expensive mistake in this area.

---

## 6. Data is the work

```text
COLLECTION      obtaining representative examples
LABELLING       usually the largest cost, in person-hours
CLEANING        duplicates, errors, inconsistent labels
BALANCING       rare classes need attention
SPLITTING       train / validation / test — chronologically for time-series
```

> [!TIP]
> **Improving the data almost always beats improving the model.** Fixing a hundred mislabelled examples typically helps more than a week of hyperparameter tuning, and teams reliably discover this in the wrong order.

---

## 7. Hyperparameters that matter

```text
LEARNING RATE   the most important single setting
                too high → diverges;  too low → never converges
BATCH SIZE      larger = more stable, more memory
EPOCHS          bounded by early stopping, not chosen in advance
```

> [!TIP]
> Start from the defaults published with the pre-trained model you are fine-tuning. They were chosen by people who trained it, and they are a far better starting point than a search over a grid you invented.

---

## 8. Reproducibility

```text
✓ Set random seeds
✓ Pin library and driver versions
✓ Version the DATASET, not just the code
✓ Log hyperparameters and metrics for every run
✓ Store the resulting artifact with its provenance
```

> [!CAUTION]
> **"Which data produced this model?" is a question you will be asked** — during an incident, an audit, or when a model behaves oddly six months later. Without dataset versioning it is unanswerable, and the model becomes untrustworthy by default.

---

## 9. Real World Example

- **Fine-tuning an image classifier** for a specific defect type on a production line, from a few hundred labelled photographs.
- **Fine-tuning a small language model** for a narrow, high-volume classification task, replacing per-call API costs.
- **LoRA adapters** for style and format, kept small enough to swap per customer.

---

## 10. Communication and Dependencies

- **A [GPU](GPU%20Computing.md)** with enough VRAM — the practical constraint
- **A pre-trained model** as the starting point
- **Labelled data**, versioned
- **Experiment tracking** — you will run dozens of variants
- **A held-out test set** touched once

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Fine-tune when you need consistent format or behaviour, when you need a small fast model for a narrow high-volume task, or when a pre-trained model is close but not adapted to your domain.

> [!CAUTION]
> - **Not to teach facts** — RAG
> - **Not with a few dozen examples** — you will overfit
> - **Not before trying prompting** — it is free and immediate
> - **Not without an evaluation set**, or you cannot tell whether it worked

---

## 12. Advantages and Disadvantages

**Advantages**
- Consistent output format and behaviour
- A small fine-tuned model is far cheaper and faster than a large general one at volume
- Runs on your own hardware — no per-call cost, no data leaving
- LoRA makes it cheap and the artifacts small

**Disadvantages**
- Requires labelled data
- Goes stale — new requirements mean retraining
- Does not reliably teach facts
- Easy to overfit on small datasets
- GPU cost and time
- Another artifact to version, deploy and monitor

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Full fine-tune** | Hours to days; substantial VRAM |
| **LoRA** | Minutes to hours; a fraction of the memory |
| **Batch size** | Limited by VRAM — the usual bottleneck |
| **Mixed precision** | Roughly halves memory, speeds training, minimal accuracy cost |
| **Data loading** | A slow pipeline starves the GPU — check utilisation |

> [!TIP]
> **If GPU utilisation is low during training, the problem is your data loader, not your model.** An idle GPU waiting for images to be decoded is one of the most common and most wasteful training inefficiencies.

---

## 14. Security Considerations

> [!CAUTION]
> **Models memorise training data and can reproduce it.** A model fine-tuned on customer emails may emit fragments of them. This makes training data a data-protection matter, not merely an engineering input — and a deletion request cannot be satisfied by deleting a row once the model has learned it.

- **Establish a lawful basis** for personal data used in training, and document it
- **Data poisoning** — anyone who can influence your training data can shape the model
- **Bias in the data becomes discriminatory behaviour**, with legal consequences in regulated domains
- **Pre-trained weights are executable artifacts** — pickle-based checkpoints from untrusted sources are remote code execution; prefer safetensors
- **Check the licence of pre-trained models** — several restrict commercial use, and this is discovered late

---

## 15. Mental Model

> [!NOTE]
> **Training from scratch is raising a child. Fine-tuning is training a graduate for your specific job.**
>
> The graduate already knows language, reasoning and general knowledge. You are teaching them your forms, your tone and your edge cases — which takes a week, not eighteen years.

---

## 16. Mini Architecture Diagram

```text
Versioned dataset
    ↓
Train / validation / test split
    ↓
Pre-trained weights → fine-tuning (or LoRA adapter)
    ↓
Validation each epoch → early stopping
    ↓
Test set, ONCE
    ↓
Model artifact + provenance → registry → serving
```

---

## 17. Complete Request Flow

```text
2,000 labelled examples collected and versioned
    ↓
Split chronologically: 70 / 15 / 15
    ↓
Pre-trained base loaded; LoRA adapter attached
    ↓
Training with mixed precision, batch size set by available VRAM
    ↓
Each epoch: validation loss measured
    ↓
Epoch 7: validation loss rises → early stopping triggers
    ↓
Best checkpoint (epoch 6) kept
    ↓
Test set evaluated ONCE → 87%
    ↓
Artifact registered with: dataset version, hyperparameters, metrics, base model
    ↓
Deployed behind a serving API
    ↓
Production accuracy monitored — it will drift
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Fine-tune from a pre-trained model, stop when validation loss turns upward, and expect the data work to dominate — training does not teach facts, it teaches behaviour.

---

## 19. Common Mistakes

- **Fine-tuning to teach knowledge** where RAG was the answer
- **Watching training loss** instead of validation loss
- **Repeatedly evaluating on the test set** until it is meaningless
- **No dataset versioning** — the model's provenance is lost
- **Training from scratch** unnecessarily
- **Low GPU utilisation** from a slow data loader
- **Ignoring the base model's licence**
- **Loading pickle checkpoints** from untrusted sources

---

## 20. Open Source Technologies

- **PyTorch**, **Hugging Face Transformers**, **PEFT** (LoRA)
- **Weights & Biases**, **MLflow** — experiment tracking and model registry
- **DVC**, **LakeFS** — dataset versioning
- **Accelerate**, **DeepSpeed** — distributed and memory-efficient training
- **safetensors** — safe weight serialisation

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Plot training and validation loss for one run and identify the correct stopping point.
- [ ] Check your GPU utilisation during training — is the data loader keeping up?
- [ ] Write down which dataset version produced your current production model.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Versioned data → pre-trained base → fine-tuning → early stopping → registry → serving
```

## 2. Request Flow

```text
Input       labelled data and a pre-trained model
    ↓
Processing  forward, loss, backward, update — until validation stops improving
    ↓
Output      a versioned model artifact with recorded provenance
```

## 3. Real-World Usage

**LoRA fine-tuning** made adaptation routine: small adapters, trained in hours on one GPU, swapped per task or per customer. It moved fine-tuning from a research activity to something a product team can do in an afternoon.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Adjusting model parameters so predictions generalise to unseen data |
| **Why does it exist?** | Because a general model rarely matches a specific task's needs |
| **Where does it belong?** | Offline, between a versioned dataset and a model registry |
| **When should I use it?** | For behaviour and format — never as a way to teach facts |

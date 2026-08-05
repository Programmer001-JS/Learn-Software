# Deep Learning

> **In one line —** stacked layers of simple maths that learn their own features, which is why it beat hand-engineered approaches at images, audio and text — and why it needs so much data and hardware.

| | |
|---|---|
| **Category** | Discipline |
| **Architectural Layer** | Data / Compute |
| **Related notes** | [Machine Learning](Machine%20Learning.md) · [Transformers](Transformers.md) · [GPU Computing](GPU%20Computing.md) · [Model Training](Model%20Training.md) · [Computer Vision](Computer%20Vision.md) |

---

## 1. Short Definition

*What is it?*

Deep learning uses **neural networks with many layers**. Each layer transforms its input, and the composition of many simple transformations can approximate extremely complex functions.

---

## 2. Purpose

*What is its main purpose?*

To learn **features automatically** rather than having humans design them — which is the whole reason it displaced classical methods for images, audio and language.

---

## 3. Problem

*What engineering problem does it solve?*

```text
CLASSICAL COMPUTER VISION             DEEP LEARNING
engineers design features             the network learns features
    edge detectors                        layer 1: edges
    corner detectors                      layer 2: textures
    colour histograms                     layer 3: shapes
    ↓                                     layer 4: objects
years of work per problem domain          ↓
brittle, does not transfer            the same architecture works for
                                      cats, tumours and defects
```

> [!IMPORTANT]
> **Automatic feature learning is the entire breakthrough.** Everything else — GPUs, more data, better optimisers — made it practical, but the reason it works at all is that nobody has to describe what a face looks like.

---

## 4. Architecture Position

```text
Data
    ↓
Framework  (PyTorch / TensorFlow)
    ↓
Neural network — layers of weights
    ↓
CUDA / ROCm
    ↓
GPU  ← matrix multiplication, at scale
```

---

## 5. How a network learns

```text
FORWARD PASS      input → layers → prediction
    ↓
LOSS              how wrong was it?
    ↓
BACKPROPAGATION   compute how each weight contributed to the error
    ↓
GRADIENT DESCENT  nudge every weight slightly in the improving direction
    ↓
Repeat, millions of times
```

That is the whole algorithm. The sophistication is in architecture, data and optimisation — not in the learning rule.

---

## 6. The architectures worth knowing

| Architecture | Built for | Status |
|---|---|---|
| **MLP** (fully connected) | Tabular, simple tasks | Rarely competitive |
| **CNN** | Images — exploits spatial locality | Still strong for vision |
| **RNN / LSTM** | Sequences | Largely superseded |
| **[Transformer](Transformers.md)** | Sequences, attention-based | **Dominant** — text, and increasingly vision |
| **Diffusion** | Image and video generation | The generative image standard |

---

## 7. Transfer learning — the practical technique

> [!TIP]
> **You will almost never train from scratch, and you should not.** Take a model already trained on an enormous dataset, and adapt it to your task with a small one.

```text
Pre-trained model (millions of images / billions of tokens)
    ↓
Freeze most layers — they already know edges, textures, grammar
    ↓
Fine-tune the last layers on YOUR few thousand examples
    ↓
Excellent results, hours instead of weeks, on one GPU
```

This is what makes deep learning accessible without a research budget. A classifier for a niche industrial defect can be built from a few hundred labelled images this way.

---

## 8. What it costs

```text
DATA        thousands to millions of labelled examples
COMPUTE     GPUs, for hours to weeks
MEMORY      model weights + activations must fit in VRAM
EXPERTISE   architecture, hyperparameters, debugging non-convergence
TIME        experiments take hours, so iteration is slow
```

> [!CAUTION]
> **Deep learning is the wrong default for tabular data.** Gradient boosting typically wins there with less data, less compute and far less tuning. Reach for neural networks when the input is an image, a waveform or natural language.

---

## 9. Overfitting and regularisation

With enough parameters a network can memorise its training set perfectly. The techniques that prevent this are standard:

```text
DROPOUT           randomly disable neurons during training
DATA AUGMENTATION rotate, crop, flip — more effective examples for free
EARLY STOPPING    stop when validation loss stops improving
WEIGHT DECAY      penalise large weights
BATCH NORM        stabilises and speeds up training
```

> [!TIP]
> **Data augmentation is usually the highest-value one for vision.** Turning 1,000 images into 10,000 varied ones costs nothing and often beats architectural changes.

---

## 10. Real World Example

- **Image classification and detection** — quality control, medical imaging, document processing.
- **Speech recognition and synthesis.**
- **[Language models](LLM%20Fundamentals.md)** — transformers scaled up.
- **Generative images and video** — diffusion models.

---

## 11. Communication and Dependencies

- **PyTorch** or **TensorFlow**
- **[GPU](GPU%20Computing.md)** with sufficient VRAM — usually the binding constraint
- **[CUDA](CUDA.md)** or ROCm drivers
- **A data pipeline** that can feed the GPU fast enough
- **Experiment tracking**, because you will run hundreds of variants

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use deep learning for images, audio, video and natural language — and start with a pre-trained model plus fine-tuning, not with training from scratch.

> [!CAUTION]
> - **Not for tabular data** — use gradient boosting
> - **Not with a few hundred examples** and no pre-trained model to adapt
> - **Not where decisions must be explained** — interpretability is genuinely poor
> - **Not when a smaller model suffices** — inference cost is permanent, training cost is once

---

## 13. Advantages and Disadvantages

**Advantages**
- Learns features automatically
- State of the art for perception and language
- Transfer learning makes it accessible with modest data
- One architecture family transfers across many problems

**Disadvantages**
- Data-hungry and compute-hungry
- Expensive inference, often requiring GPUs in production
- Poorly interpretable
- Hard to debug — a failure to converge has many possible causes
- Sensitive to hyperparameters
- Large models are large deployment artifacts

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Training** | GPU-hours to GPU-weeks |
| **Inference** | Milliseconds to seconds; often needs a GPU |
| **VRAM** | The hard constraint — "CUDA out of memory" is the defining error |
| **Batching** | Processing 32 inputs together costs barely more than one |
| **Quantisation** | Reducing precision shrinks and speeds models with modest accuracy loss |

> [!TIP]
> **Batching is the single most important serving optimisation.** A GPU processing one request at a time is mostly idle; the same GPU with batched requests can serve many times the throughput at nearly the same latency.

---

## 15. Security Considerations

> [!CAUTION]
> **Adversarial examples** are a genuine and unsolved problem: imperceptible changes to an image can flip a classification with high confidence. Any deep learning system making security-relevant decisions from untrusted input needs defence in depth, not just a model.

- **Models memorise training data** — a network trained on personal data can leak it
- **Pre-trained weights from the internet are executable artifacts**; loading a pickled PyTorch checkpoint from an untrusted source is remote code execution. Prefer safetensors
- **Data poisoning** — influencing training data shapes the model
- **Bias** — trained on historical data, models reproduce historical patterns, including unlawful ones

---

## 16. Mental Model

> [!NOTE]
> **A deep network is a stack of increasingly abstract filters.**
>
> The first layer sees edges. The next combines edges into textures, then into shapes, then into objects. Nobody described any of those stages — the network discovered that this hierarchy is a useful way to compress the problem, because the data forced it to.

---

## 17. Mini Architecture Diagram

```text
Input (image / text / audio)
    ↓
Layer 1  → low-level features
Layer 2  → mid-level features
Layer N  → task-specific features
    ↓
Output layer → prediction
    ↑
Backpropagation adjusts every weight
```

---

## 18. Complete Request Flow

```text
─────────── training (transfer learning) ───────────
Pre-trained backbone loaded
    ↓
Early layers frozen — they already know edges and textures
    ↓
Final layers replaced for your classes
    ↓
Your 2,000 images, augmented into effectively 20,000
    ↓
Forward → loss → backward → update, for a few epochs on one GPU
    ↓
Early stopping when validation loss plateaus
    ↓
─────────── serving ───────────
Request arrives
    ↓
Batched with other pending requests           ← the key optimisation
    ↓
Preprocessed identically to training           ← skew here silently ruins accuracy
    ↓
Tensor copied to VRAM
    ↓
Forward pass on the GPU
    ↓
Probabilities returned; thresholded into a decision
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> Deep learning learns its own features, which is why it dominates images, audio and language — and transfer learning is what makes it usable without a research budget.

---

## 20. Common Mistakes

- **Using it for tabular data** where boosting is better
- **Training from scratch** when a pre-trained model exists
- **Preprocessing differently** at training and serving time
- **No data augmentation** on a small image dataset
- **Not batching** at inference, wasting most of the GPU
- **Loading untrusted model checkpoints**
- **Ignoring VRAM limits** until deployment
- **Treating confidence scores as calibrated probabilities** — they usually are not

---

## 21. Open Source Technologies

- **PyTorch**, **TensorFlow**, **JAX** — frameworks
- **Hugging Face Transformers**, **timm** — pre-trained models
- **safetensors** — a safe model serialisation format
- **ONNX Runtime**, **TensorRT** — optimised inference
- **Weights & Biases**, **MLflow** — experiment tracking
- **Albumentations** — image augmentation

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Fine-tune a pre-trained image model on a small dataset of your own and compare it with training from scratch.
- [ ] Measure your inference throughput with batch size 1 versus 32.
- [ ] Check whether your serving preprocessing is identical to your training preprocessing.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Data → framework → layered network → CUDA → GPU
```

## 2. Request Flow

```text
Input       an image, waveform or token sequence
    ↓
Processing  transformed layer by layer into increasingly abstract features
    ↓
Output      a prediction, ideally computed in a batch
```

## 3. Real-World Usage

**Transfer learning** is what brought deep learning into ordinary engineering. A team with a few hundred labelled images and one GPU can build a working classifier — which was impossible when every model had to be trained from nothing.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Neural networks with many layers that learn features automatically |
| **Why does it exist?** | Because hand-designed features could not handle perception |
| **Where does it belong?** | On GPUs, behind a batched serving layer |
| **When should I use it?** | Images, audio and language — starting from a pre-trained model |

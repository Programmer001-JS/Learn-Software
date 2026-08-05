# GPU Computing

> [!NOTE]
> **This note is the AI-engineering half of a pair.**
> **[02 - GPU](../02%20-%20Computer%20Science%20Fundamentals/GPU.md)** covers the hardware: what a GPU is, SIMT, why thousands of simple cores beat a few complex ones. **Read that first.**
> **This note** covers what matters once you are actually running AI workloads on one: VRAM as the real constraint, utilisation, cost, and how not to waste an expensive machine.

> **In one line —** the constraint is never the compute; it is VRAM, and then it is keeping the GPU fed.

| | |
|---|---|
| **Category** | Operational Practice |
| **Architectural Layer** | Infrastructure |
| **Related notes** | [GPU](../02%20-%20Computer%20Science%20Fundamentals/GPU.md) · [CUDA](CUDA.md) · [Model Serving](Model%20Serving.md) · [Model Training](Model%20Training.md) · [Cloud Fundamentals](../12%20-%20Cloud%20Architecture/Cloud%20Fundamentals.md) |

---

## 1. VRAM is the constraint

```text
"Will this model run?"
    ↓
NOT "is the GPU fast enough?"
    ↓
"Does it fit in VRAM?"
```

```text
Rough memory budget for inference:
    model weights
  + activations (scales with batch size and sequence length)
  + framework overhead
    ↓
Exceed it → CUDA out of memory. Not slower — it simply fails.
```

> [!IMPORTANT]
> **"CUDA out of memory" is the defining error of this field.** A GPU with more raw compute but less VRAM is useless for a model that does not fit. When choosing hardware, VRAM capacity is the first specification to check, not the last.

---

## 2. Precision — the main lever

```text
float32   4 bytes/parameter    the training default
float16   2 bytes              half the memory, standard for inference
bfloat16  2 bytes              better numerical range; preferred where supported
int8      1 byte               quantised; small accuracy loss
int4      0.5 bytes            aggressive; noticeable loss on some tasks
```

> [!TIP]
> **Quantisation is usually the difference between "needs an expensive GPU" and "runs on one you already have."** Moving from float32 to int8 cuts memory roughly fourfold, and for many inference workloads the accuracy cost is small enough to be irrelevant.

---

## 3. Utilisation — the second constraint

```text
nvidia-smi shows 20% GPU utilisation
    ↓
You are paying for an idle machine
    ↓
The cause is almost never the GPU:
    slow data loading (disk, decoding, preprocessing)
    batch size of 1
    CPU-bound preprocessing
    synchronous host↔device copies
```

> [!IMPORTANT]
> **A GPU at low utilisation is a data pipeline problem.** Before renting a bigger one, check whether the current one is actually working — this single check has saved a great deal of money in a great many projects.

---

## 4. The transfer cost

```text
System RAM  ──PCIe──►  VRAM
                ↑
        This copy is often the bottleneck for small workloads
```

```text
Small tensor, frequent transfers  → the copy dominates; CPU may be faster
Large batch, few transfers        → the GPU wins decisively
```

> [!TIP]
> **Keep data on the GPU between operations.** Moving a tensor back to the CPU to run one small operation and then returning it is a common and expensive pattern — visible as low utilisation and high latency with no obvious cause.

---

## 5. Cost

```text
GPU instances cost roughly 10–40× a comparable CPU instance
    ↓
An idle GPU costs the same as a busy one
    ↓
Which makes utilisation a financial metric, not just a technical one
```

```text
Practical controls:
    batch requests                  → far more work per GPU-hour
    quantise                        → a smaller, cheaper GPU suffices
    spot / preemptible instances    → substantial discount for interruptible training
    scale to zero for sporadic work → accept the cold start
    separate training and serving   → different hardware, different lifetimes
```

> [!CAUTION]
> **Forgotten GPU instances are the most common large cloud bill surprise.** A training job left running over a weekend, or an autoscaled inference fleet with no upper bound, produces invoices that get noticed at the wrong level of the organisation.

---

## 6. Sharing a GPU

```text
NAIVE           several processes on one GPU → they compete for VRAM
                → one allocates too much, everything crashes

MPS             concurrent kernels from multiple processes
MIG             the GPU partitioned into isolated instances (data-centre cards)
BATCHING        one process, many requests            ← usually the right answer
```

> [!TIP]
> For inference, **one process with dynamic batching almost always beats several processes sharing a card.** It uses the hardware better and removes the VRAM contention entirely.

---

## 7. Training versus serving

| | Training | Serving |
|---|---|---|
| **Duration** | Hours to days | Continuous |
| **VRAM need** | High — gradients and optimiser state | Lower — weights and activations |
| **Interruptible** | Yes, with checkpointing → **use spot instances** | No |
| **Utilisation target** | Near 100% | Depends on traffic |
| **Scaling** | Bigger or more GPUs | More replicas |

> [!TIP]
> Training is interruptible if you checkpoint, which makes **spot instances a large and underused saving**. Serving is not, so it belongs on reliable capacity.

---

## 8. Real World Example

- **Quantising a model to int8** so it fits on a mid-range GPU instead of a data-centre card — the single most common cost optimisation.
- **Dynamic batching** in inference servers, turning 5% utilisation into 70%.
- **Spot instances with checkpointing** for training, at a fraction of on-demand cost.
- **Edge inference** on small accelerators, removing the cloud GPU entirely.

---

## 9. Communication and Dependencies

- **[CUDA](CUDA.md)** or ROCm drivers, matched to the framework version
- **A framework** — PyTorch, TensorFlow, ONNX Runtime, TensorRT
- **Container images with the right driver stack** — a frequent source of deployment friction
- **Monitoring** — `nvidia-smi`, DCGM, or your platform's GPU metrics

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Use a GPU for training anything non-trivial, for real-time vision or video, and for serving models where CPU latency is unacceptable.

> [!CAUTION]
> - **Not for small models at low volume** — CPU inference is often adequate and vastly cheaper
> - **Not for tabular ML** — gradient boosting on CPU is usually faster end to end
> - **Not before measuring** — many teams rent a GPU and discover their bottleneck was preprocessing

---

## 11. Advantages and Disadvantages

**Advantages**
- Orders of magnitude faster on parallel numeric work
- Makes training and large-model inference feasible at all
- High memory bandwidth
- Mature framework support

**Disadvantages**
- **VRAM is the hard limit**
- Expensive, and equally expensive when idle
- Driver and version compatibility is a recurring operational cost
- Vendor lock-in through CUDA in practice
- Poor at branching and sequential logic
- Scarce and sometimes unavailable in a given region

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **VRAM** | Determines feasibility, not just speed |
| **Batch size** | The dominant throughput lever |
| **Precision** | Halving or quartering memory, with modest accuracy cost |
| **Data pipeline** | The usual cause of low utilisation |
| **PCIe transfers** | Dominant for small, frequent operations |

---

## 13. Security Considerations

> [!CAUTION]
> **GPU memory is not reliably cleared between processes on all platforms.** On shared or rented GPU infrastructure, data from a previous workload can potentially be read by the next — a weaker isolation story than CPU virtualisation offers.

- **Model weights in VRAM are extractable** with sufficient access; they are valuable intellectual property
- **Multi-tenant GPU hosting** deserves scrutiny; MIG provides genuine partitioning where available
- **An unauthenticated inference endpoint on GPU hardware** is a cost-exhaustion attack with an unusually high per-request price
- **Driver vulnerabilities** exist and are patched; GPU drivers are privileged kernel code

---

## 14. Mental Model

> [!NOTE]
> **A GPU is a factory floor with a thousand workers and one small loading bay.**
>
> The workers are extremely fast. The constraint is how much material fits inside (VRAM) and how quickly you can get it through the bay (PCIe). Hiring more workers helps nothing if the bay is the bottleneck — and an idle factory costs the same as a busy one.

---

## 15. Mini Architecture Diagram

```text
Data pipeline (CPU)  ← usually the bottleneck
    ↓ PCIe transfer
VRAM
  ├─ model weights (quantised)
  ├─ activations (scale with batch size)
  └─ framework overhead
    ↓
GPU cores — batched work
    ↓ PCIe transfer back
Results
```

---

## 16. Complete Request Flow

```text
Serving deployment
    ↓
Model quantised to int8 → fits on a mid-range GPU
    ↓
Loaded into VRAM once at startup
    ↓
Requests arrive → dynamic batching queue
    ↓
Batch of 32 preprocessed ON THE CPU — in parallel, ahead of time
    ↓
One PCIe transfer for the whole batch
    ↓
GPU forward pass — utilisation ~70%
    ↓
One transfer back; results split per request
    ↓
─────────── diagnosing a slow deployment ───────────
nvidia-smi shows 15% utilisation
    ↓
NOT a GPU problem
    ↓
Image decoding on a single CPU thread was starving it
    ↓
Parallel decoding added → utilisation 70%, throughput 4× — same hardware
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> VRAM decides what runs, batching decides throughput, and low utilisation is almost always a data pipeline problem rather than a reason to rent a bigger GPU.

---

## 18. Common Mistakes

- **Choosing a GPU by compute rather than VRAM**
- **Batch size of 1** in serving
- **Blaming the GPU** for a CPU-bound data pipeline
- **Not quantising** before scaling hardware
- **Leaving instances running** — the classic cloud bill incident
- **On-demand instances for interruptible training** instead of spot
- **Frequent small host↔device transfers**
- **Mismatched driver, CUDA and framework versions**

---

## 19. Open Source Technologies

- **PyTorch**, **TensorFlow**, **JAX**
- **bitsandbytes**, **GPTQ**, **AWQ** — quantisation
- **vLLM**, **TensorRT**, **ONNX Runtime** — optimised inference
- **nvidia-smi**, **DCGM**, **nvtop** — monitoring
- **NVIDIA Container Toolkit** — GPUs inside containers
- **ROCm** — AMD's alternative to CUDA

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Check GPU utilisation during your heaviest workload. If it is below 50%, find out why.
- [ ] Calculate your model's VRAM requirement at float32 and at int8.
- [ ] Check whether any GPU instance in your account is running without a current workload.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
CPU data pipeline → PCIe → VRAM (weights + activations) → GPU cores → results
```

## 2. Request Flow

```text
Input       a batch of work, prepared on the CPU
    ↓
Processing  transferred once, executed in parallel on thousands of cores
    ↓
Output      results returned — bounded by VRAM and by how fast you can feed it
```

## 3. Real-World Usage

**Quantisation plus dynamic batching** is the standard cost optimisation in production inference. Together they routinely turn a deployment that needed several expensive GPUs into one that runs comfortably on a single mid-range card.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Running AI workloads on GPU hardware, in production |
| **Why does it matter?** | Because VRAM, batching and utilisation decide feasibility and cost |
| **Where does it belong?** | Beneath training and serving, as the resource they compete for |
| **When should I use it?** | Training, real-time vision, and inference where CPU latency is unacceptable |

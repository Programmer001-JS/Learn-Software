# GPU

> **In one line —** thousands of simple cores that do the same arithmetic on thousands of numbers at once; useless for logic, unbeatable for maths.

| | |
|---|---|
| **Full name** | Graphics Processing Unit |
| **Category** | Hardware Component |
| **Architectural Layer** | Physical / Compute |
| **Related notes** | [CPU](CPU.md) · [RAM](RAM.md) · [GPU Computing](../11%20-%20AI%20Engineering/GPU%20Computing.md) · [CUDA](../11%20-%20AI%20Engineering/CUDA.md) · [Model Training](../11%20-%20AI%20Engineering/Model%20Training.md) |

---

## 1. Short Definition

*What is it?*

A GPU is a processor built from thousands of small, simple cores designed to run the **same operation on many pieces of data simultaneously**. It was created to draw pixels, and turned out to be exactly the right shape for neural networks.

---

## 2. Purpose

*What is its main purpose?*

To perform massively parallel arithmetic — mainly matrix and vector operations — far faster than a [CPU](CPU.md) can. Graphics, video encoding, scientific simulation and all modern AI are the same underlying problem: enormous numbers of independent multiplications and additions.

---

## 3. Problem

*What engineering problem does it solve?*

A CPU has a few powerful cores optimised for complicated, branching, sequential work. Rendering a screen means computing millions of pixels that do not depend on each other, and training a model means multiplying enormous matrices.

```text
CPU    8–64 cores, each very smart      →  great at "do many different things"
GPU    5,000–20,000 cores, each simple  →  great at "do one thing 20,000 times"
```

Doing the second kind of work on a CPU wastes almost all of the available hardware.

---

## 4. Architecture Position

The GPU is a **co-processor**. The CPU stays in charge and hands it batches of parallel work.

```text
Application code
    ↓
Framework  (PyTorch, TensorFlow, OpenGL, ffmpeg)
    ↓
CUDA / ROCm / Metal driver
    ↓
CPU  ──── copies data ────►  GPU VRAM
                                ↓
                          thousands of cores
                                ↓
CPU  ◄──── copies result ────  GPU VRAM
```

> [!IMPORTANT]
> That copy between system RAM and GPU VRAM is a real cost, and it is the most common reason GPU code turns out slower than expected.

---

## 5. Real World Example

- **OpenAI, Anthropic, Google** — train and serve large language models on GPU clusters; there is no CPU-only path to a modern model.
- **YouTube / Netflix** — hardware-accelerated video encoding and transcoding.
- **Tesla / autonomous driving** — real-time [computer vision](../11%20-%20AI%20Engineering/Computer%20Vision.md) on every frame from every camera.
- **Games** — the original purpose: transform geometry and shade millions of pixels, 60+ times per second.

---

## 6. Input

*What does it receive as input?*

- **Data** — arrays, matrices, tensors, textures, copied into the GPU's own VRAM
- **A kernel** — a small program that will be executed by every core, on its own slice of the data

---

## 7. Processing

*What happens inside it?*

The GPU launches thousands of threads, each running the *same* kernel on a different element. Threads are grouped so that a whole group executes in lockstep.

```text
Data:    [ a1  a2  a3  a4  ...  a20000 ]
              ↓   ↓   ↓   ↓          ↓
Cores:      c1  c2  c3  c4   ...  c20000
              ↓   ↓   ↓   ↓          ↓
Result:  [ r1  r2  r3  r4  ...  r20000 ]
```

This model is called **SIMT** — Single Instruction, Multiple Threads.

---

## 8. Output

*What does it return?*

The transformed data in VRAM: a rendered frame, a batch of model activations, an encoded video segment. It must usually be copied back to system RAM before the CPU can use it.

---

## 9. Internal Idea

*How does it work internally?*

The GPU trades **cleverness for count**. Each core is far simpler than a CPU core: weak branch prediction, small cache, no complex out-of-order machinery. In exchange, you get thousands of them plus very high memory bandwidth.

The consequence matters: if threads in the same group take *different* branches of an `if`, the hardware must run both paths and discard half the work. Branching kills GPU performance in a way it never does on a CPU.

---

## 10. Communication

- **[CPU](CPU.md)** — over PCIe (or a unified memory bus on Apple Silicon)
- **VRAM** — its own dedicated, very high-bandwidth memory
- **Other GPUs** — via NVLink or the network, for multi-GPU training
- **Frameworks** — PyTorch, TensorFlow, ONNX Runtime, ffmpeg, OpenGL/Vulkan

---

## 11. Dependencies

- **Drivers** — CUDA (NVIDIA), ROCm (AMD), Metal (Apple)
- **Enough VRAM** to hold the model or dataset batch
- **Substantial power and cooling**
- A framework that supports the hardware; portability across vendors is still poor

---

## 12. Alternatives

```text
CPU        general purpose; fine for small models and all sequential logic
    ↓
GPU        the practical default for AI and graphics
    ↓
TPU        Google's chip, built specifically for tensor operations
    ↓
NPU        on-device AI accelerators in phones and laptops
    ↓
FPGA/ASIC  fixed-function hardware; fastest and least flexible
```

---

## 13. When To Use

> [!TIP]
> Use a GPU when the same operation is applied independently to thousands of items: model training and inference, image and video processing, simulation, large matrix maths.

---

## 14. When NOT To Use

> [!CAUTION]
> A GPU will not help — and often hurts — when:
> - The work is sequential or full of branching logic
> - The dataset is small; the copy to VRAM costs more than the computation saves
> - You are I/O-bound, which describes almost every ordinary web application
> - The model fits comfortably on a CPU and latency is already acceptable
>
> GPU instances in the cloud cost roughly 10–40× a comparable CPU instance. An idle GPU is an expensive space heater.

---

## 15. Advantages

- Orders of magnitude faster on parallel numeric work
- Very high memory bandwidth
- Mature ecosystem for AI and graphics
- Makes otherwise impossible workloads (large model training) feasible at all

---

## 16. Disadvantages

- Expensive to buy and to rent
- **VRAM is limited** and is usually the binding constraint on model size
- Data transfer between RAM and VRAM is a frequent bottleneck
- Poor at branching and sequential logic
- Vendor lock-in: CUDA is NVIDIA-only in practice

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | 10–100× on suitable workloads; slower than CPU on unsuitable ones |
| **Memory** | VRAM is separate and scarce — "CUDA out of memory" is the defining error message |
| **CPU** | Still needed to orchestrate; a slow data pipeline starves the GPU |
| **GPU** | Utilisation is the metric to watch; low utilisation means the pipeline is the bottleneck |
| **Network** | Multi-GPU training is often limited by interconnect bandwidth |

---

## 18. Security Considerations

> [!CAUTION]
> GPU memory is **not reliably cleared** between processes on all platforms, so data from a previous workload can potentially be read by the next one — a genuine concern on shared or rented GPU infrastructure.

- Model weights in VRAM are valuable intellectual property and are extractable with sufficient access
- Multi-tenant GPU hosting has a weaker isolation story than CPU virtualisation

---

## 19. Mental Model

> [!NOTE]
> **CPU = a few PhD mathematicians. GPU = twenty thousand schoolchildren with calculators.**
>
> Give the mathematicians a complex proof and they will solve it. Give them twenty thousand multiplications and they will be slower than the children — who cannot do anything else, but can all multiply at the same time.

---

## 20. Mini Architecture Diagram

```text
Application
    ↓
PyTorch / TensorFlow / ffmpeg
    ↓
CUDA driver
    ↓
CPU  ──copy──►  VRAM  ──►  thousands of GPU cores
                  ↑                    ↓
                  └──── results ───────┘
    ↓
CPU reads the result back
```

---

## 21. Complete Request Flow

Serving one AI inference request:

```text
HTTP request with input text
    ↓
API server (CPU) tokenises the input
    ↓
Tensor copied from RAM to VRAM
    ↓
GPU runs the model: thousands of cores do matrix multiplication
    ↓
Output tensor copied back to RAM
    ↓
CPU decodes it into text
    ↓
HTTP response
```

> [!IMPORTANT]
> Requests are usually **batched** before reaching the GPU, because processing 32 inputs together costs barely more than processing one. This is why AI serving systems care so much about batching.

---

## 22. Key Takeaway

> [!IMPORTANT]
> A GPU is fast only when the same operation is applied to thousands of independent items at once — for anything else, the CPU wins.

---

## 23. Common Mistakes

- **Assuming a GPU makes everything faster** — it only helps parallel arithmetic
- **Ignoring the RAM↔VRAM copy**, which can dominate the runtime for small jobs
- **Running out of VRAM** and not understanding that it is separate from system memory
- **Leaving GPU instances running** — the single most common cloud cost disaster
- **Not batching inference requests**, which wastes most of the hardware
- **Confusing training with inference** — their hardware needs are very different

---

## 24. Open Source Technologies

- **PyTorch**, **TensorFlow**, **JAX** — GPU-accelerated ML frameworks
- **ONNX Runtime**, **vLLM**, **llama.cpp** — model serving and inference
- **ROCm** — AMD's open alternative to CUDA
- **OpenCL**, **Vulkan**, **Triton** — vendor-neutral GPU programming

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Take one task from your project and decide honestly whether it is parallel enough to benefit from a GPU.
- [ ] Look up the VRAM required for a model you are interested in, and compare it with a GPU you could actually afford.
- [ ] Explain in two sentences why branching code performs badly on a GPU.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Application
    ↓
CPU  (orchestration, data preparation)
    ↓
GPU  (parallel computation)  ←→  VRAM
    ↓
Result back to CPU
```

## 2. Request Flow

```text
Input       a batch of data copied into VRAM
    ↓
Processing  thousands of cores run the same kernel in parallel
    ↓
Output      transformed data copied back to system RAM
```

## 3. Real-World Usage

Every large language model in production, including this one, is trained and served on GPU clusters. The parallel structure of matrix multiplication is the entire reason modern AI became practical when it did.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A processor with thousands of simple cores for parallel arithmetic |
| **Why does it exist?** | Because CPUs waste most of their capacity on massively parallel maths |
| **Where does it belong?** | Beside the CPU, as a co-processor with its own memory |
| **When should I use it?** | When the same operation runs on thousands of independent items |

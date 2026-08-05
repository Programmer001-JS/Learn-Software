# CUDA

> **In one line —** NVIDIA's platform for running general computation on GPUs; you will almost never write it, and you will regularly fight its version compatibility.

| | |
|---|---|
| **Full name** | Compute Unified Device Architecture |
| **Category** | Parallel Computing Platform |
| **Architectural Layer** | Driver / Runtime |
| **Vendor** | NVIDIA |
| **Related notes** | [GPU Computing](GPU%20Computing.md) · [GPU](../02%20-%20Computer%20Science%20Fundamentals/GPU.md) · [Deep Learning](Deep%20Learning.md) · [Docker](../13%20-%20DevOps%20and%20Delivery/Docker.md) |

---

## 1. Short Definition

*What is it?*

CUDA is NVIDIA's programming model and runtime for general-purpose GPU computation. It provides the layer between frameworks such as PyTorch and the GPU hardware itself.

---

## 2. Purpose

*What is its main purpose?*

To let ordinary programs use the GPU for parallel computation. Before CUDA, using a GPU for anything other than graphics meant expressing your problem as a shader — which almost nobody did.

---

## 3. Why it matters to you

> [!IMPORTANT]
> **You will not write CUDA kernels.** PyTorch, TensorFlow and ONNX Runtime do that. What you will do is deal with CUDA as an **operational dependency** — versions, drivers, container images, and the errors that arise when they disagree.

That is the honest framing of this note: CUDA is infrastructure you configure, not a language you learn.

---

## 4. Architecture Position

```text
Your Python code
    ↓
PyTorch / TensorFlow
    ↓
cuDNN, cuBLAS  (optimised primitives)
    ↓
CUDA RUNTIME    ← the version your framework was built against
    ↓
CUDA DRIVER     ← the version installed on the host
    ↓
GPU hardware
```

> [!CAUTION]
> **Those two middle layers are different things, and confusing them causes most CUDA problems.** The *driver* is installed on the host and must be new enough. The *runtime* ships with your framework or container. A driver older than the runtime fails; a newer driver is generally fine.

---

## 5. The version compatibility problem

```text
PyTorch built for CUDA 12.1
    ↓
Host driver supports up to CUDA 11.8
    ↓
"CUDA driver version is insufficient for CUDA runtime version"
```

```text
The pieces that must line up:
    GPU architecture (compute capability)
    NVIDIA driver version
    CUDA runtime version
    cuDNN version
    Framework build
```

> [!TIP]
> **Use official framework container images.** They pin the runtime, cuDNN and framework together, correctly. Installing CUDA manually and matching versions by hand is a solved problem you do not need to solve again — and it is where a surprising number of days are lost.

---

## 6. Containers and GPUs

```text
A normal container cannot see the GPU
    ↓
NVIDIA Container Toolkit passes the driver through
    ↓
docker run --gpus all ...
```

```text
Inside the container:  CUDA runtime + framework
On the host:           the NVIDIA driver
    ↓
The container does NOT contain a driver — it borrows the host's
```

> [!IMPORTANT]
> This split is why a container that works on one machine fails on another with an older driver. The image is portable; the host driver requirement travels with it implicitly.

---

## 7. The errors you will actually see

```text
"CUDA out of memory"
    → VRAM exhausted. Reduce batch size, quantise, or free cached tensors.
      See GPU Computing — this is the most common error in the field.

"no kernel image is available for execution on the device"
    → the framework build does not support your GPU's compute capability

"CUDA driver version is insufficient"
    → host driver too old for the runtime in the container

"CUDA error: device-side assert triggered"
    → usually an out-of-range index; rerun on CPU for a readable stack trace
```

> [!TIP]
> **Reproduce on CPU to debug.** GPU errors are frequently reported asynchronously and at the wrong line. The same code on CPU gives a clear Python traceback pointing at the actual bug.

---

## 8. What CUDA actually provides

```text
KERNELS         functions executed by thousands of threads in parallel
MEMORY MODEL    global, shared and register memory, explicitly managed
STREAMS         concurrent execution and overlapping transfers
LIBRARIES       cuBLAS (linear algebra) · cuDNN (neural networks)
                cuFFT · NCCL (multi-GPU communication)
```

The libraries are the important part: **frameworks get their speed from cuDNN and cuBLAS**, not from CUDA code written by framework authors.

---

## 9. The lock-in question

```text
CUDA    NVIDIA only — the de facto standard, best supported
ROCm    AMD's equivalent — improving, less mature ecosystem
Metal   Apple Silicon — good for local development
oneAPI  Intel
OpenCL  vendor-neutral, largely superseded in practice
Triton  a Python DSL that compiles to GPU kernels; reduces CUDA-specific code
```

> [!CAUTION]
> **CUDA is genuine vendor lock-in**, and it is the main reason NVIDIA's position in AI is as strong as it is. Frameworks abstract most of it — PyTorch code often runs unchanged on ROCm — but optimised kernels, quantisation libraries and inference servers frequently assume CUDA. Factor that into hardware decisions rather than discovering it during a migration.

---

## 10. Real World Example

- **Every PyTorch GPU workload** runs through CUDA, whether you notice or not.
- **Container images** shipping a specific CUDA runtime are the standard deployment unit for AI services.
- **cuDNN version mismatches** are a recurring cause of "it works on my machine".
- **Multi-GPU training** uses NCCL, CUDA's collective communication library.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> You use CUDA whenever you use an NVIDIA GPU. The practical decision is *how* — and the answer is nearly always: use an official container image and do not install it by hand.

> [!CAUTION]
> Writing custom CUDA kernels is justified only when profiling shows a specific operation dominating and no library primitive covers it. For almost all application work, that threshold is never reached — and **Triton** offers most of the benefit in Python.

---

## 12. Advantages and Disadvantages

**Advantages**
- The most mature GPU compute ecosystem by a wide margin
- Highly optimised libraries underpin every major framework
- Excellent tooling and profilers
- Universally supported by AI software

**Disadvantages**
- **NVIDIA-only** — real vendor lock-in
- Version compatibility is a persistent operational cost
- Opaque error messages
- Driver dependency complicates containerisation
- Kernel-level programming has a steep learning curve

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **cuDNN / cuBLAS** | Where framework speed actually comes from |
| **Streams** | Overlap transfers with compute — real gains, rarely used directly |
| **Version** | Newer runtimes frequently include meaningful optimisations |
| **Mixed precision** | Tensor cores, exposed through the framework |

---

## 14. Security Considerations

> [!CAUTION]
> **GPU drivers are privileged kernel code**, and CUDA drivers have published vulnerabilities. On a machine running untrusted workloads, an unpatched driver is a privilege-escalation path — and driver updates are frequently deferred because they require a reboot and disrupt training jobs.

- **VRAM is not reliably zeroed between processes** on all configurations — relevant on shared infrastructure
- **`--gpus all` grants a container access to the GPU**, and with it whatever isolation weaknesses exist
- **Keep drivers patched**, and plan for the reboot rather than avoiding it indefinitely

---

## 15. Mental Model

> [!NOTE]
> **CUDA is the electrical wiring of the building.**
>
> You do not think about it while the lights work. You think about it constantly on the day the new appliance needs a different voltage than the wiring supplies — which, in this analogy, is roughly every framework upgrade.

---

## 16. Mini Architecture Diagram

```text
Python / PyTorch
    ↓
cuDNN · cuBLAS · NCCL
    ↓
CUDA runtime      ← in your container image
    ↓
CUDA driver       ← on the host
    ↓
GPU
```

---

## 17. Complete Request Flow

Deploying a GPU service:

```text
Choose an official framework image with a pinned CUDA runtime
    ↓
Verify the host driver version meets the runtime's requirement
    ↓
Install the NVIDIA Container Toolkit on the host
    ↓
docker run --gpus all
    ↓
Framework detects the GPU; cuDNN and cuBLAS initialise
    ↓
Model loaded into VRAM
    ↓
Inference runs — cuBLAS handles the matrix multiplication
    ↓
─────────── failure mode ───────────
Deployed to an older node → "driver version is insufficient"
    ↓
Container is correct; the HOST is the problem
    ↓
Fix: pin node driver versions, or constrain scheduling to compatible nodes
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> CUDA is an operational dependency rather than something you write — use official container images, keep the host driver current, and account for the vendor lock-in when choosing hardware.

---

## 19. Common Mistakes

- **Installing CUDA manually** and matching versions by hand
- **Confusing driver and runtime versions**
- **Assuming a container carries its own driver**
- **Debugging device-side asserts on GPU** instead of reproducing on CPU
- **Ignoring lock-in** until a migration is proposed
- **Deferring driver patches indefinitely**
- **Writing custom kernels** before profiling proves the need

---

## 20. Open Source Technologies

- **CUDA Toolkit**, **cuDNN**, **NCCL** — NVIDIA's stack
- **NVIDIA Container Toolkit** — GPUs in containers
- **Triton** (OpenAI) — GPU kernels written in Python
- **ROCm** — AMD's alternative
- **Nsight Systems / Compute** — profiling
- **PyTorch / TensorFlow official images** — the recommended way to get a working stack

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Run `nvidia-smi` and note the driver version, then check the CUDA runtime your framework was built against.
- [ ] Reproduce a GPU error on CPU and compare how readable the traceback is.
- [ ] Check whether your deployment pins host driver versions or assumes they are compatible.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Framework → cuDNN/cuBLAS → CUDA runtime (container) → CUDA driver (host) → GPU
```

## 2. Request Flow

```text
Input       a tensor operation from a framework
    ↓
Processing  dispatched to an optimised library kernel via the CUDA runtime
    ↓
Output      results in VRAM — provided every version in the chain agrees
```

## 3. Real-World Usage

**Official framework container images** exist because CUDA version matching is difficult enough that framework maintainers decided to solve it once for everyone. Using them is the single most effective way to avoid this entire category of problem.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | NVIDIA's GPU compute platform, beneath every AI framework |
| **Why does it exist?** | To make general computation on GPUs practical |
| **Where does it belong?** | Between your framework and the GPU driver |
| **When should I use it?** | Always, indirectly — configured through container images, not written |

# ONNX Runtime

> **In one line —** train in any framework, export to one portable format, and run it fast anywhere — without shipping PyTorch to production.

| | |
|---|---|
| **Full name** | Open Neural Network Exchange Runtime |
| **Category** | Inference Engine |
| **Architectural Layer** | Model serving |
| **Related notes** | [Model Serving](Model%20Serving.md) · [Deep Learning](Deep%20Learning.md) · [GPU Computing](GPU%20Computing.md) · [CUDA](CUDA.md) |

---

## 1. Short Definition

*What is it?*

**ONNX** is an open format for representing trained models as a computation graph. **ONNX Runtime** is an engine that executes those graphs efficiently across CPUs, GPUs and specialised accelerators.

---

## 2. Purpose

*What is its main purpose?*

To separate **where a model is trained** from **where it runs** — and to make inference faster than the training framework can.

---

## 3. Problem

*What engineering problem does it solve?*

```text
TRAINING FRAMEWORK IN PRODUCTION      ONNX RUNTIME
ship PyTorch: ~2 GB of dependencies   ship a runtime: tens of MB
Python required                       C++, C#, Java, JavaScript, Rust
optimised for FLEXIBILITY             optimised for INFERENCE
    ↓                                     ↓
large images, slow start              small images, fast start
one language                          deploy anywhere
```

> [!IMPORTANT]
> **Training frameworks are built for experimentation, not for serving.** PyTorch keeps autograd machinery, dynamic graph construction and debugging support that inference does not need and pays for. ONNX Runtime discards all of it.

---

## 4. Architecture Position

```text
PyTorch / TensorFlow / scikit-learn
    ↓  export
model.onnx  — a portable computation graph
    ↓
ONNX RUNTIME
    ├─ graph optimisation (fusion, constant folding, dead node removal)
    └─ execution provider: CPU · CUDA · TensorRT · CoreML · DirectML
    ↓
Hardware
```

---

## 5. Where the speed comes from

```text
GRAPH OPTIMISATION
    operator fusion       conv + bias + ReLU → one kernel
    constant folding      compute what is knowable ahead of time
    dead node elimination remove training-only branches
    layout optimisation   arrange memory for the target hardware
    ↓
Typically 2–5× faster than the training framework, on identical hardware
```

> [!TIP]
> **This is free performance.** No accuracy loss, no code change beyond the export step. Teams routinely scale hardware before trying this, and it is one of the cheapest wins available in [model serving](Model%20Serving.md).

---

## 6. Execution providers

```text
CPUExecutionProvider        everywhere; surprisingly good with quantisation
CUDAExecutionProvider       NVIDIA GPUs
TensorRTExecutionProvider   NVIDIA, fastest, more export friction
CoreMLExecutionProvider     Apple devices
DirectMLExecutionProvider   Windows, any vendor
OpenVINOExecutionProvider   Intel CPUs and accelerators
```

The same `.onnx` file runs on all of them. Choosing a provider is a deployment configuration, not a code change.

---

## 7. Quantisation

```text
Dynamic quantisation   post-export, no calibration data needed, immediate
Static quantisation    uses representative data, better accuracy, more work
    ↓
float32 → int8:  ~4× smaller, ~2–4× faster on CPU
```

> [!TIP]
> **Quantised ONNX on CPU frequently removes the need for a GPU entirely** for small and medium models. That changes the cost of a service by an order of magnitude, and it is worth testing before provisioning GPU inference.

---

## 8. Where export goes wrong

> [!CAUTION]
> Export is the friction point, and it fails in predictable ways.

```text
UNSUPPORTED OPERATOR     a custom or exotic layer has no ONNX equivalent
DYNAMIC CONTROL FLOW     Python if/loops depending on tensor VALUES do not trace
DYNAMIC SHAPES           must be declared explicitly at export time
PREPROCESSING            usually NOT exported — it stays in your code
NUMERICAL DRIFT          small differences from operator implementations
```

> [!IMPORTANT]
> **Always validate the exported model against the original** on a real sample: run both, compare outputs numerically, and check accuracy on your evaluation set. A model that exports without error can still behave differently — and a silent 3% accuracy loss is worse than a failed export.

---

## 9. Real World Example

- **Edge and mobile deployment** — the same model on a server, a phone and a browser.
- **CPU-only inference** in environments where GPUs are unavailable or unjustified.
- **.NET and Java services** running models without a Python process.
- **Browser inference** via ONNX Runtime Web, keeping data on the user's device.
- **Reducing container images** from gigabytes to tens of megabytes.

---

## 10. Communication and Dependencies

- **An exporter** — `torch.onnx.export`, `tf2onnx`, `skl2onnx`
- **The runtime** for your language and platform
- **An execution provider** matched to the hardware
- **Preprocessing code**, reimplemented consistently at the target

> [!CAUTION]
> That last point causes real bugs. Exporting the model but reimplementing preprocessing in another language is exactly the [training/serving skew](Machine%20Learning.md) problem — and it produces quietly worse results with no error.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use ONNX Runtime when deploying to a language other than Python, to edge or mobile devices, to CPU-only environments, or when you want faster inference and smaller images without changing the model.

> [!CAUTION]
> - **Not during development** — iterate in the training framework
> - **Not for models using unsupported operators** without checking exportability first
> - **Not for large language models** without checking — dedicated servers such as vLLM handle those better
> - **Not without validating** the exported model's outputs

---

## 12. Advantages and Disadvantages

**Advantages**
- One format, many platforms and languages
- 2–5× faster inference through graph optimisation
- Small deployment artifacts
- Quantisation support, often removing the GPU requirement
- Vendor-neutral, with an open specification

**Disadvantages**
- Export can fail on unsupported or dynamic operations
- Preprocessing is not included
- An extra build step and artifact to manage
- Debugging an exported graph is harder than debugging Python
- Opset version compatibility between exporter and runtime

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Inference speed** | Typically 2–5× over the training framework |
| **With quantisation** | A further 2–4× on CPU |
| **Image size** | Tens of MB versus gigabytes |
| **Cold start** | Substantially faster — relevant for serverless |
| **Memory** | Lower, with no training machinery loaded |

---

## 14. Security Considerations

> [!IMPORTANT]
> **ONNX is a genuine security improvement over pickle-based checkpoints.** A PyTorch `.pt` file loaded with `torch.load` can execute arbitrary code; an `.onnx` file is a protobuf computation graph, not executable Python. For models obtained from outside your organisation, this matters.

- **Validate models from untrusted sources regardless** — malformed protobuf has its own parsing risks
- **The model is still intellectual property** — an ONNX file is straightforward to inspect and reuse
- **Smaller attack surface in production** — no Python interpreter and no training framework in the image
- **Pin the runtime version**; it is a dependency like any other

---

## 15. Mental Model

> [!NOTE]
> **Training frameworks are a workshop; ONNX Runtime is the finished product in a shipping box.**
>
> The workshop has every tool, jigs, offcuts and space to change your mind. None of that goes to the customer. You export the finished object, and it is smaller, faster and works in places the workshop could never fit.

---

## 16. Mini Architecture Diagram

```text
PyTorch model
    ↓ export + validate
model.onnx
    ↓
ONNX Runtime
  ├─ graph optimisation
  ├─ optional quantisation
  └─ execution provider (CPU / CUDA / TensorRT / CoreML)
    ↓
Server · mobile · browser · embedded
```

---

## 17. Complete Request Flow

```text
Model trained in PyTorch
    ↓
Exported with explicit dynamic axes for batch size
    ↓
VALIDATED: same inputs → outputs compared numerically against PyTorch
    ↓
Evaluation set re-run → accuracy unchanged
    ↓
Quantised to int8; accuracy re-checked → 0.4% loss, accepted
    ↓
Deployed in a 40 MB container with ONNX Runtime, CPU provider
    ↓
─────────── serving ───────────
Request arrives
    ↓
Preprocessing — SAME implementation as training
    ↓
Batched, run through the optimised graph
    ↓
~12 ms on CPU, where PyTorch on CPU took ~45 ms
    ↓
No GPU needed; the service costs a fraction of the GPU deployment
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Exporting to ONNX gives faster inference, smaller images and cross-platform deployment for free — but validate the exported model's outputs, and remember preprocessing does not come with it.

---

## 19. Common Mistakes

- **Not validating** the exported model against the original
- **Assuming preprocessing is included**
- **Forgetting dynamic axes**, fixing the batch size at export
- **Opset version mismatch** between exporter and runtime
- **Quantising without re-measuring accuracy**
- **Exporting a model with unsupported operators** and discovering it late
- **Using it during development** instead of at deployment

---

## 20. Open Source Technologies

- **ONNX**, **ONNX Runtime** — the format and the engine
- **ONNX Runtime Web**, **ONNX Runtime Mobile** — browser and device
- **Netron** — visualise an ONNX graph; genuinely useful for debugging exports
- **TensorRT**, **OpenVINO**, **CoreML** — execution providers
- **Optimum** (Hugging Face) — export transformers to ONNX

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Export one model to ONNX and compare inference time against the training framework.
- [ ] Compare outputs numerically between the original and the export on 100 samples.
- [ ] Quantise it and measure whether CPU inference is now fast enough to skip the GPU.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Training framework → ONNX export → optimisation → execution provider → hardware
```

## 2. Request Flow

```text
Input       a preprocessed tensor
    ↓
Processing  executed through an optimised, fused computation graph
    ↓
Output      predictions, faster and in a smaller footprint than the framework
```

## 3. Real-World Usage

**Browser inference with ONNX Runtime Web** runs models on the user's device, which removes both the server cost and the data-protection question of sending input anywhere. One export format making that possible alongside server deployment is the clearest demonstration of the format's value.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A portable model format and an optimised inference engine |
| **Why does it exist?** | Because training frameworks are the wrong tool for serving |
| **Where does it belong?** | Between training and deployment |
| **When should I use it?** | Cross-platform, CPU-only, edge, or when you want free speed |

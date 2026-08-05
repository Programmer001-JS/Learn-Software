# Model Serving

> **In one line —** running a trained model as a production service, where the model is the easy part and batching, memory and versioning are the actual work.

| | |
|---|---|
| **Category** | Practice |
| **Architectural Layer** | Application / Infrastructure |
| **Related notes** | [Model Training](Model%20Training.md) · [GPU Computing](GPU%20Computing.md) · [ONNX Runtime](ONNX%20Runtime.md) · [AI Pipelines](AI%20Pipelines.md) · [Backend Fundamentals](../06%20-%20Backend%20Architecture/Backend%20Fundamentals.md) |

---

## 1. Short Definition

*What is it?*

Model serving is exposing a trained model as an API that other systems call — with the operational concerns of any service, plus large memory requirements, GPU dependencies and non-deterministic output.

---

## 2. Problem

*What engineering problem does it solve?*

```text
A model in a notebook
    ↓
Loaded once, called by one person, no error handling
    ↓
Production needs:
    concurrency · latency targets · versioning · rollback
    monitoring · resource limits · graceful degradation
```

> [!IMPORTANT]
> **A model is a service with unusual resource requirements.** Everything from folders 06, 13 and 14 applies unchanged — timeouts, health checks, rate limits, graceful shutdown. Teams frequently treat model serving as a separate discipline and rediscover ordinary backend engineering the hard way.

---

## 3. Architecture Position

```text
Application
    ↓
API gateway / load balancer
    ↓
Model serving layer
    ├─ request batching     ← the largest single optimisation
    ├─ model in memory / VRAM
    └─ preprocessing (identical to training)
    ↓
GPU or CPU
```

---

## 4. Batching is the main lever

```text
NO BATCHING                           DYNAMIC BATCHING
one request at a time                 wait ~10 ms, collect what arrives
    ↓                                     ↓
GPU mostly idle                       process 32 together
~50 requests/second                   ~800 requests/second
                                          ↓
                                      +10 ms latency, 16× throughput
```

> [!IMPORTANT]
> **A GPU processing one request at a time is being wasted.** Its advantage is parallelism, and a batch of one uses almost none of it. Dynamic batching — collect requests for a few milliseconds, run them together — is the single highest-value change in most serving deployments.

---

## 5. Cold starts and model loading

```text
Container starts
    ↓
Load weights from disk               seconds to minutes for large models
    ↓
Copy to VRAM
    ↓
Warm-up inference                    first call is always slow
    ↓
Ready
```

> [!CAUTION]
> **Autoscaling a model service is far slower than autoscaling a web service.** A traffic spike that a stateless API absorbs in seconds may take minutes for a model service. Keep a warm baseline, scale on a leading indicator such as queue depth, and never expect scale-to-zero to be free.

---

## 6. Deployment options

```text
EMBEDDED IN THE APP     simplest; the model's memory is your app's memory
                        → small models, low volume

DEDICATED SERVICE       scale independently, share across applications
                        → the usual production shape

INFERENCE SERVER        Triton, TorchServe, vLLM, KServe
                        → batching, versioning and metrics built in

SERVERLESS              scale to zero; cold starts hurt
                        → sporadic, latency-tolerant workloads

MANAGED API             no infrastructure; per-call cost, data leaves
                        → fastest to start
```

---

## 7. Optimise before scaling

```text
1. QUANTISATION      float32 → int8: smaller, faster, small accuracy loss
2. ONNX / TensorRT   graph optimisation; often 2–5× faster
3. DISTILLATION      train a small model to imitate a large one
4. CACHING           identical inputs need not be recomputed
5. Only then: more hardware
```

> [!TIP]
> **Quantisation and graph compilation are frequently worth more than a bigger GPU**, and they cost nothing per month. Teams routinely scale hardware before trying either.

---

## 8. Versioning and rollback

```text
Model artifacts are versioned like code:
    v1.2.0 in production
    v1.3.0 shadow-deployed — receives traffic, results not used
    ↓
Compare outputs on real traffic
    ↓
Canary 5% → monitor → full rollout
    ↓
Regression → roll back to v1.2.0 immediately
```

> [!IMPORTANT]
> **A model change is a production change and needs the same discipline as a code deploy.** Shadow deployment is particularly valuable here, because model quality regressions are not caught by tests — they only appear on real traffic.

---

## 9. Monitoring

```text
Standard service metrics: latency, throughput, errors, saturation
    ↓
PLUS model-specific:
    prediction distribution      has the output shifted?
    input distribution           has the world changed?  ← drift
    confidence distribution      is the model less sure than it was?
    fallback rate                how often is the model unavailable?
```

> [!CAUTION]
> **A model degrades silently.** No error is raised when accuracy drops from 88% to 71% — the API still returns 200. Without distribution monitoring, the first signal is a business metric moving in the wrong direction weeks later.

---

## 10. Real World Example

- **Recommendation and ranking services** — high volume, tight latency budgets, heavy batching.
- **Vision inference** behind an upload pipeline, processed asynchronously.
- **Language model serving** with vLLM, where continuous batching and KV caching dominate throughput.
- **Fraud scoring** in the request path, with a strict timeout and a rules-based fallback.

---

## 11. Communication and Dependencies

- **A [GPU](GPU%20Computing.md)** for larger models; CPU is often adequate for small ones
- **A model registry** — artifacts with provenance
- **A serving framework** for batching, versioning and metrics
- **Preprocessing code shared with training**
- **A fallback path** for when the model is unavailable or too slow

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Serve a model as a dedicated service when several applications use it, when it needs GPU hardware the rest of your system does not, or when it must scale independently.

> [!CAUTION]
> Embedding a small model directly in your application is entirely legitimate and often better — a service boundary you do not need is latency and operational cost you did not have to pay.
>
> And **never put a slow model in a synchronous request path without a timeout and a fallback.** Model latency is variable; a request that waits indefinitely for inference is a hang.

---

## 13. Advantages and Disadvantages

**Advantages**
- Independent scaling and hardware allocation
- One model shared by many consumers
- Versioning and rollback separated from application deploys
- Batching becomes possible across callers

**Disadvantages**
- Network hop and serialisation cost
- Slow cold starts and slow autoscaling
- Expensive hardware, often idle
- Non-deterministic output complicates testing
- Silent quality degradation

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Batching** | The dominant throughput factor |
| **Quantisation** | 2–4× faster, several times smaller |
| **ONNX / TensorRT** | Frequently 2–5× over the training framework |
| **VRAM** | Determines how many models and how large a batch |
| **Cold start** | Seconds to minutes — plan capacity accordingly |

---

## 15. Security Considerations

> [!CAUTION]
> **Model artifacts are executable.** Loading a pickle-based checkpoint runs arbitrary code. A model registry is a software supply chain, and treating downloaded weights as inert data is a genuine vulnerability — prefer safetensors and verify provenance.

- **Inference endpoints are expensive per request** — an unauthenticated one is a cost-exhaustion vector; authenticate and rate-limit
- **Input validation applies** — malformed images and oversized payloads reach native decoders
- **Model outputs are untrusted** downstream; never pass them into a query, shell or eval unvalidated
- **Model extraction** — an attacker with enough queries can approximate your model; rate limiting is the practical mitigation
- **Inputs sent to a managed API leave your infrastructure**

---

## 16. Mental Model

> [!NOTE]
> **A model server is a specialist consultant who is expensive, occasionally slow, and sometimes wrong.**
>
> You book them properly (batching), you give them a deadline (timeout), you have a plan if they are unavailable (fallback), and you check their work (monitoring). What you do not do is put them on the critical path with no time limit and assume they will always answer.

---

## 17. Mini Architecture Diagram

```text
Clients
    ↓
Load balancer
    ↓
Model service replicas
  ├─ dynamic batching queue
  ├─ preprocessing (shared with training)
  ├─ model in VRAM (quantised, ONNX)
  └─ metrics: latency · distributions · fallback rate
    ↓
GPU
    ↓
Fallback path when unavailable or timed out
```

---

## 18. Complete Request Flow

```text
Request arrives
    ↓
Authenticated and rate-limited
    ↓
Input validated
    ↓
Placed in the BATCHING QUEUE — waits up to 10 ms
    ↓
Batch of 32 assembled
    ↓
Preprocessed identically to training
    ↓
Copied to VRAM; forward pass; results split back per request
    ↓
Response in ~40 ms total
    ↓
Prediction and input features logged for drift monitoring
    ↓
─────────── degraded ───────────
GPU saturated → latency exceeds the timeout
    ↓
FALLBACK: a rules-based score, or a cached result, or an explicit "unavailable"
    ↓
The application continues; it does not hang
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> Batching is the largest serving optimisation, model quality degrades without raising errors, and every synchronous inference call needs a timeout and a fallback.

---

## 20. Common Mistakes

- **No batching** — most of the GPU wasted
- **No timeout or fallback** in a synchronous path
- **Monitoring latency but not prediction distributions**
- **Deploying a model without shadow or canary evaluation**
- **Preprocessing that differs from training**
- **Scaling hardware before quantising or compiling**
- **Loading untrusted pickle checkpoints**
- **Unauthenticated inference endpoints**

---

## 21. Open Source Technologies

- **vLLM**, **TGI** — language model serving with continuous batching
- **Triton Inference Server**, **TorchServe**, **KServe** — general model serving
- **[ONNX Runtime](ONNX%20Runtime.md)**, **TensorRT**, **OpenVINO** — optimised inference
- **BentoML**, **Ray Serve** — packaging and serving frameworks
- **Evidently**, **NannyML** — drift detection
- **safetensors** — safe model loading

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Measure your throughput with and without dynamic batching.
- [ ] Check whether every synchronous inference call has a timeout and a defined fallback.
- [ ] Add prediction-distribution monitoring and observe it over a week.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Client → load balancer → batching queue → model (GPU) → response + metrics
```

## 2. Request Flow

```text
Input       a validated inference request
    ↓
Processing  batched, preprocessed consistently, executed on GPU
    ↓
Output      a prediction within the latency budget, or a fallback
```

## 3. Real-World Usage

**vLLM's continuous batching** demonstrates the principle at its extreme: by batching at the token level rather than the request level, it achieves several times the throughput of naive serving on identical hardware. Batching, not hardware, is where the capacity is.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Running a trained model as a production service |
| **Why does it exist?** | Because a model in a notebook is not a system |
| **Where does it belong?** | Behind a load balancer, with batching and a fallback |
| **When should I use it?** | When several consumers need the model, or it needs its own hardware |

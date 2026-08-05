# AI Pipelines

> **In one line —** the automated path from raw data to a monitored model in production, which is the difference between a notebook that worked once and a system that keeps working.

| | |
|---|---|
| **Category** | Practice |
| **Architectural Layer** | Data / Infrastructure |
| **Also called** | MLOps |
| **Related notes** | [Model Training](Model%20Training.md) · [Model Serving](Model%20Serving.md) · [CI CD](../13%20-%20DevOps%20and%20Delivery/CI%20CD.md) · [Machine Learning](Machine%20Learning.md) · [Monitoring and Alerting](../14%20-%20Scalability%20and%20Reliability/Monitoring%20and%20Alerting.md) |

---

## 1. Short Definition

*What is it?*

An AI pipeline is the automated sequence that takes raw data through preparation, training, evaluation, deployment and monitoring — reproducibly, and repeatedly.

---

## 2. Problem

*What engineering problem does it solve?*

```text
A notebook produced a good model
    ↓
Nobody knows which data version, which library versions, which parameters
    ↓
Six months later, accuracy has quietly fallen
    ↓
Nobody can retrain it, because nobody can reproduce it
```

> [!IMPORTANT]
> **The model is not the deliverable — the pipeline that produces it is.** A model is a snapshot that begins going stale immediately. Without a reproducible path to make a new one, you have a liability rather than an asset.

---

## 3. Architecture Position

```text
Data sources
    ↓
INGESTION → validation
    ↓
FEATURE ENGINEERING  (shared between training and serving)
    ↓
TRAINING → EVALUATION → gate
    ↓
Model registry (versioned artifacts + provenance)
    ↓
DEPLOYMENT (shadow → canary → full)
    ↓
MONITORING → drift detected → back to training
```

---

## 4. What makes it different from a normal CI/CD pipeline

```text
SOFTWARE CI/CD                        AI PIPELINE
code changes                          code AND data AND model change
tests pass or fail                    metrics improve or regress — a threshold
deterministic builds                  stochastic training
version the code                      version code, data, model, features
deploy on merge                       deploy on a quality gate
works or breaks                       DEGRADES SILENTLY
```

> [!IMPORTANT]
> **Three versioned things instead of one.** "Which model is in production?" must be answerable as: this code, this dataset version, these hyperparameters, this evaluation result. Anything less makes incidents unresolvable.

---

## 5. The stages

| Stage | Purpose |
|---|---|
| **Ingestion** | Collect raw data, on a schedule or as events |
| **Validation** | Schema, ranges, nulls, distribution — **fail loudly** |
| **Feature engineering** | Transform into model inputs |
| **Training** | Produce a candidate model |
| **Evaluation** | Compare against the current production model |
| **Gate** | Automated pass/fail on defined metrics |
| **Registry** | Store the artifact with full provenance |
| **Deployment** | Shadow, canary, then full rollout |
| **Monitoring** | Drift, prediction distribution, business metrics |

---

## 6. Data validation is the highest-value stage

```text
Upstream changes a column from euros to cents
    ↓
No error anywhere — the numbers are still numbers
    ↓
Model retrains on data 100× off
    ↓
Deployed, silently wrong for weeks
```

> [!CAUTION]
> **Garbage in, garbage deployed.** Validating schema, ranges and statistical distributions before training is the cheapest defence in the entire pipeline, and it is the stage most often skipped because it feels like it is not doing anything.

---

## 7. The training/serving skew problem

```text
TRAINING                              SERVING
features computed in pandas           features computed in application code
offline, batch                        online, per request
    ↓                                     ↓
subtly different values → the model performs worse in production than in evaluation,
                          with no bug anywhere
```

**The fixes, in order of robustness:**

```text
1. FEATURE STORE     one definition, served to both paths
2. SHARED CODE       the same library imported by both
3. DISCIPLINE        two implementations kept in step  ← will drift
```

---

## 8. The quality gate

```text
New model trained
    ↓
Evaluated on the SAME held-out set as the current production model
    ↓
Better on the primary metric?          → and not worse on any guardrail metric?
    ↓ YES                                  ↓ NO
promote to shadow                      stop; alert; do not deploy
```

> [!TIP]
> **Guardrail metrics matter as much as the primary one.** A model with better overall accuracy that is markedly worse for one customer segment is a regression, not an improvement — and only a guardrail check catches it.

---

## 9. Deployment strategy

```text
SHADOW      runs on real traffic; predictions logged, not used
            → compare against production on identical inputs
    ↓
CANARY      5% of traffic
            → watch business metrics, not just accuracy
    ↓
FULL        with the previous version retained for rollback
```

Model quality regressions do not appear in tests. Shadow deployment is the only way to compare two models on the traffic that actually exists.

---

## 10. Monitoring — what to watch

```text
INPUT DRIFT        has the incoming data distribution changed?
PREDICTION DRIFT   has the output distribution changed?
CONFIDENCE         is the model less certain than it was?
BUSINESS METRIC    the one that actually matters
GROUND TRUTH LAG   labels may arrive days or weeks later
```

> [!CAUTION]
> **Ground truth lag is what makes AI monitoring genuinely hard.** You cannot measure accuracy today for a churn model whose outcome is known in ninety days. Drift detection on inputs and predictions is the *leading* indicator you have; accuracy is the lagging one.

---

## 11. Retraining

```text
SCHEDULED     weekly or monthly — simple, may be unnecessary or too late
TRIGGERED     when drift crosses a threshold — responsive, needs monitoring
CONTINUOUS    online learning — powerful, and easy to poison
```

> [!TIP]
> **Start with scheduled retraining and a manual gate.** Fully automatic retrain-and-deploy loops can degrade quickly if data quality slips — the pipeline faithfully learns whatever went wrong upstream.

---

## 12. Real World Example

- **Recommendation systems** — retrained frequently, with heavy A/B evaluation.
- **Fraud models** — drift is adversarial and continuous, because attackers adapt deliberately.
- **Demand forecasting** — seasonal retraining with explicit distribution checks.
- **The common failure**: a model deployed once, monitored never, quietly degrading for a year.

---

## 13. When To Use / When NOT To Use

> [!TIP]
> Build the pipeline as soon as a model reaches production and matters. Even a minimal version — versioned data, a reproducible training script, an evaluation gate and drift monitoring — prevents the situations described above.

> [!CAUTION]
> Do not build a full MLOps platform for one model that is retrained twice a year. A scheduled script, a versioned dataset and a dashboard may be entirely sufficient — and a platform nobody uses is a maintenance cost with no return.

---

## 14. Advantages and Disadvantages

**Advantages**
- Reproducible: the same inputs produce the same model
- Retraining is routine rather than a project
- Regressions caught before users see them
- Provenance answers "why did the model do that?"
- Drift detected before the business metric moves

**Disadvantages**
- Real infrastructure investment
- More moving parts than a training script
- Versioning data is genuinely harder than versioning code
- Tooling in this space is immature and changes rapidly
- Easy to over-engineer

---

## 15. Performance Impact

| Aspect | Impact |
|---|---|
| **Training runs** | GPU time on a schedule — a recurring cost |
| **Feature computation** | Frequently the slowest stage |
| **Validation** | Cheap, and the highest return per unit of effort |
| **Shadow deployment** | Doubles inference cost during the comparison |

---

## 16. Security Considerations

> [!CAUTION]
> **An automated retraining pipeline is a supply chain into your production model.** Anyone who can influence the training data can influence the model's behaviour — and with automatic deployment, without a human ever reading the data.

- **Validate and monitor incoming data** as a security control, not only a quality one
- **Access control on the model registry** — a swapped artifact is a swapped model
- **Provenance for every artifact**: which data, which code, which run
- **Personal data used in training** requires a lawful basis, and models memorise
- **Keep a human gate** on deployment for anything consequential; fully automatic retrain-and-deploy removes the last check
- **Audit trail** — for regulated domains, "why did the model decline this application?" must be answerable

---

## 17. Mental Model

> [!NOTE]
> **A model is bread; the pipeline is the bakery.**
>
> A good loaf is not a business. Being able to bake the same loaf tomorrow, notice when the flour changed, and stop a bad batch before it reaches the shelf — that is the business. Teams routinely celebrate the loaf and never build the bakery.

---

## 18. Mini Architecture Diagram

```text
Sources → ingestion → VALIDATION (fail loudly)
    ↓
Feature engineering ──shared──► serving path
    ↓
Training → evaluation vs production → GATE
    ↓
Model registry (artifact + data version + code version + metrics)
    ↓
Shadow → canary → full  (previous version retained)
    ↓
Monitoring: input drift · prediction drift · business metric
    ↓
Threshold crossed → retrain
```

---

## 19. Complete Request Flow

```text
Weekly schedule fires
    ↓
Fresh data ingested
    ↓
VALIDATION: schema, ranges, distribution vs last week
    ↓ a numeric column shifted by two orders of magnitude
PIPELINE STOPS — alert raised, no training run
    ↓  ← this is the pipeline earning its cost
Upstream bug fixed; rerun
    ↓
Features computed with the SHARED library
    ↓
Model trained; dataset version recorded
    ↓
Evaluated against production on the same held-out set
    ↓
Primary metric +1.2%; guardrail metrics unchanged → GATE PASSES
    ↓
Registered with full provenance
    ↓
Shadow-deployed for 24 hours; outputs compared on real traffic
    ↓
Canary at 5%; business metric watched
    ↓
Full rollout; previous version kept for rollback
    ↓
Monitoring continues → drift threshold crossed in six weeks → cycle repeats
```

---

## 20. Key Takeaway

> [!IMPORTANT]
> The pipeline is the deliverable, not the model — version data alongside code, validate inputs before training, gate on evaluation, and monitor drift because degradation is silent.

---

## 21. Common Mistakes

- **No data validation** — training on corrupted inputs
- **Data not versioned**, so the model cannot be reproduced
- **Training/serving skew** from duplicated feature code
- **No evaluation gate** — a worse model deployed automatically
- **No shadow or canary stage**
- **Monitoring errors but not drift**
- **Fully automatic retrain-and-deploy** with no human gate
- **Building a platform** for a model retrained twice a year

---

## 22. Open Source Technologies

- **MLflow**, **Weights & Biases** — experiment tracking and model registry
- **DVC**, **LakeFS** — data versioning
- **Airflow**, **Prefect**, **Dagster**, **Kubeflow** — orchestration
- **Feast** — feature store
- **Great Expectations**, **Pandera** — data validation
- **Evidently**, **NannyML** — drift detection
- **Seldon**, **KServe**, **BentoML** — deployment

---

## 23. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 24. Workbook Exercise

- [ ] For your production model, write down: which data version, which code commit, which hyperparameters. Can you?
- [ ] Add one data validation check that would have caught a real past incident.
- [ ] Set up drift monitoring on one input feature and observe it for a month.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Data → validation → features → training → gate → registry → deployment → monitoring
                                                                    ↑          │
                                                                    └── drift ─┘
```

## 2. Request Flow

```text
Input       fresh data and a training trigger
    ↓
Processing  validated, transformed, trained, evaluated against production
    ↓
Output      a versioned model, deployed gradually and monitored for drift
```

## 3. Real-World Usage

**Fraud detection pipelines** retrain continuously because the adversary adapts deliberately. They are also the clearest demonstration of why drift monitoring is not optional: a model that was excellent last month is being actively worked around this month.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The automated, reproducible path from data to a monitored production model |
| **Why does it exist?** | Because models degrade silently and notebooks cannot be reproduced |
| **Where does it belong?** | Around every model that matters |
| **When should I use it?** | As soon as a model reaches production — sized to the model's actual importance |

# Machine Learning

> **In one line —** fitting a function to examples so it generalises to data it has never seen; everything else is detail.

| | |
|---|---|
| **Category** | Discipline |
| **Architectural Layer** | Data / Application |
| **Related notes** | [AI Fundamentals](AI%20Fundamentals.md) · [Deep Learning](Deep%20Learning.md) · [Model Training](Model%20Training.md) · [Model Serving](Model%20Serving.md) · [AI Pipelines](AI%20Pipelines.md) |

---

## 1. Short Definition

*What is it?*

Machine learning is the practice of learning a function from labelled or unlabelled examples, such that it performs well on **new** data — not on the data it was trained with.

---

## 2. Purpose

*What is its main purpose?*

To make predictions where the underlying rule is unknown, too complex to write, or changes over time.

---

## 3. The central problem — generalisation

```text
UNDERFITTING              GOOD FIT                OVERFITTING
model too simple          captures the pattern    memorised the data
    ↓                         ↓                       ↓
bad on training           good on training        PERFECT on training
bad on new data           good on new data        bad on new data
```

> [!IMPORTANT]
> **A model that scores perfectly on its training data has usually learned nothing useful.** It memorised the answers. The entire discipline is about the gap between training performance and real-world performance — everything else is a technique for narrowing it.

---

## 4. The data split

```text
TRAINING SET    60–80%   the model learns from this
VALIDATION SET  10–20%   tune hyperparameters against this
TEST SET        10–20%   touched ONCE, at the very end
```

> [!CAUTION]
> **Every time you look at the test set and change something, it becomes part of your training process.** After twenty iterations of "check the test score, adjust, repeat", the test score is optimistic and no longer predicts real performance. This is the most common form of self-deception in ML work.

---

## 5. Data leakage — the failure that produces impossible results

```text
Model achieves 99.8% accuracy
    ↓
Investigation: the feature "account_closed_date" was in the training data
    ↓
It is only populated AFTER the outcome you are predicting
    ↓
The model learned to read the future
    ↓
In production: useless
```

> [!TIP]
> **Suspiciously good results are almost always leakage, not brilliance.** Other common forms: scaling before splitting the data, time-series data split randomly instead of chronologically, or duplicate rows appearing in both training and test sets.

---

## 6. The workflow

```text
1. Define the problem and the metric      ← what does "good" mean, in business terms?
    ↓
2. Collect and clean data                 ← most of the work
    ↓
3. Split: train / validation / test
    ↓
4. Feature engineering
    ↓
5. Baseline model                         ← simplest possible, always
    ↓
6. Train, evaluate, iterate
    ↓
7. Deploy
    ↓
8. Monitor for drift → back to 2
```

> [!TIP]
> **Always build the stupid baseline first**: predict the average, predict the most common class, use a linear model. If your sophisticated model does not clearly beat it, the problem is the data, not the algorithm. This step is skipped constantly and would save enormous amounts of time.

---

## 7. Algorithm families

| Family | Use for | Note |
|---|---|---|
| **Linear / logistic regression** | Baselines, interpretable models | Start here |
| **Decision trees** | Interpretable rules | Overfit alone |
| **Random forest** | General tabular data | Strong default |
| **Gradient boosting** (XGBoost, LightGBM) | **Tabular data** | Usually the winner |
| **k-means, DBSCAN** | Clustering | Unsupervised |
| **[Neural networks](Deep%20Learning.md)** | Images, text, audio | Overkill for tabular |

> [!IMPORTANT]
> **For tabular business data, gradient boosting beats deep learning in most cases** — with less data, less compute and less tuning. Deep learning dominates images, text and audio. Choosing a neural network for a spreadsheet is a common and expensive mistake.

---

## 8. Feature engineering

```text
Raw:        timestamp = 2026-08-05T14:23:00
Features:   hour_of_day = 14 · day_of_week = 2 · is_weekend = false
                is_business_hours = true
```

The model cannot infer that 23:00 and 00:00 are adjacent, or that December and January are neighbouring months. Encoding domain knowledge into features is frequently worth more than any change of algorithm.

---

## 9. Real World Example

- **Churn prediction, credit scoring, demand forecasting, fraud detection** — the highest-value applications, and almost all tabular.
- **Recommendation systems** — matrix factorisation and gradient boosting long before neural approaches.
- **Kaggle competitions on tabular data** are won overwhelmingly by gradient boosting, not deep learning.

---

## 10. Communication and Dependencies

- **A data pipeline** — the model is only as current as its inputs
- **A feature store**, in larger systems, so training and serving compute features identically
- **[Model serving](Model%20Serving.md)** infrastructure
- **Monitoring** for drift and for input distribution changes

---

## 11. Training/serving skew

```text
TRAINING                              SERVING
features computed in a notebook       features computed in application code
pandas, offline, batch                different library, online, per request
    ↓                                     ↓
subtly different results → the model performs worse in production than in testing
```

> [!CAUTION]
> This is one of the most common and most confusing production problems: the model was fine in evaluation and is worse in reality, with no bug anywhere. The fix is shared feature code, or a feature store that serves both paths.

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use ML when you have enough labelled examples, the pattern is genuinely complex, and an approximate answer is acceptable.

> [!CAUTION]
> Do not use it when you have little data, when a rule would work, when every decision must be explainable, or when the cost of being wrong is unbounded. And do not start a project without knowing **what metric would make it a success** — a model with no defined target is a research project pretending to be a deliverable.

---

## 13. Advantages and Disadvantages

**Advantages**
- Solves problems where rules cannot be written
- Improves as more data arrives
- Finds patterns humans would not notice
- Adapts to changing conditions through retraining

**Disadvantages**
- Requires substantial, well-labelled data
- Probabilistic — always wrong sometimes
- Hard to debug; failures are statistical rather than logical
- Degrades silently as the world changes
- Explainability is limited, and legally required in some domains

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Training** | Minutes to days; usually offline and batch |
| **Inference** | Milliseconds for tabular models; heavier for neural networks |
| **Memory** | Gradient boosting models are small; neural networks are not |
| **Retraining** | An ongoing operational cost, not a one-off |

---

## 15. Security Considerations

> [!CAUTION]
> **Training data is a liability as well as an asset.** Models can memorise and reproduce individual training examples, which means a model trained on personal data may leak it — and "delete my data" cannot be satisfied by deleting the row if the model has already learned it.

- **Data poisoning** — an attacker who can influence training data can shape the model's behaviour
- **Model inversion and membership inference** — an attacker can sometimes determine whether a specific record was in the training set
- **Bias is a security and legal issue**, not only an ethical one: a model trained on historical data reproduces historical discrimination, and in regulated domains that is unlawful
- **Adversarial inputs** — small, deliberate perturbations can flip a prediction
- **Never train on personal data without a lawful basis**, and document what was used

---

## 16. Mental Model

> [!NOTE]
> **Training a model is teaching by example rather than by instruction.**
>
> Show a child ten thousand photographs of dogs and they will recognise a dog they have never seen. Show them ten thousand photographs where every dog happens to be on grass, and they will learn "grass" — confidently, and without being able to tell you that is what happened.

---

## 17. Mini Architecture Diagram

```text
Raw data
    ↓
Cleaning · feature engineering
    ↓
Train / validation / test split
    ↓
Training → validation tuning → test ONCE
    ↓
Model artifact
    ↓
Serving API
    ↓
Monitoring: drift, input distribution, prediction distribution
    ↓
Retraining
```

---

## 18. Complete Request Flow

```text
─────────── training ───────────
Data collected and cleaned
    ↓
Split chronologically (not randomly, for time-series)
    ↓
Baseline model trained → 71% accuracy
    ↓
Gradient boosting → 84%
    ↓
Suspiciously high result? → check for leakage
    ↓
Validation used for tuning; test set touched once → 83%
    ↓
─────────── serving ───────────
Request arrives
    ↓
Features computed with THE SAME CODE used in training
    ↓
Model predicts: 0.87 probability
    ↓
Business threshold applied: > 0.8 → flag for review
    ↓
Outcome recorded → becomes training data for the next iteration
    ↓
─────────── six months later ───────────
Input distribution has shifted; accuracy has quietly fallen to 74%
    ↓
Monitoring catches it → retrain
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> The goal is performance on data the model has never seen — build a baseline first, guard the test set, and suspect leakage whenever the results look too good.

---

## 20. Common Mistakes

- **No baseline**, so nobody knows whether the model adds value
- **Repeatedly evaluating on the test set** until it is meaningless
- **Data leakage** producing impossible accuracy
- **Random splits on time-series data**
- **Optimising accuracy** on an imbalanced problem
- **Training/serving skew** from duplicated feature code
- **Deep learning on tabular data** where boosting is better and cheaper
- **No drift monitoring** — the model degrades silently

---

## 21. Open Source Technologies

- **scikit-learn** — the standard for classical ML
- **XGBoost**, **LightGBM**, **CatBoost** — gradient boosting
- **pandas**, **Polars** — data manipulation
- **MLflow**, **Weights & Biases** — experiment tracking
- **Feast** — feature store, addressing training/serving skew
- **Evidently**, **NannyML** — drift monitoring

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Take a dataset and build the simplest possible baseline before anything else.
- [ ] Check one existing model for leakage: is any feature unavailable at prediction time?
- [ ] Write down the business metric your model is meant to improve — not the accuracy figure.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Data → features → training → model → serving → monitoring → retraining
```

## 2. Request Flow

```text
Input       features computed from a request
    ↓
Processing  the learned function is applied
    ↓
Output      a probability, thresholded into a business decision
```

## 3. Real-World Usage

**Gradient boosting on tabular data** quietly powers most production ML in business: churn, credit, demand, fraud. It receives a fraction of the attention that deep learning does and produces a large share of the value.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Learning a function from examples that generalises to new data |
| **Why does it exist?** | Because some patterns can be demonstrated but not specified |
| **Where does it belong?** | Behind a serving API, fed by a monitored data pipeline |
| **When should I use it?** | Sufficient data, genuine complexity, tolerance for being wrong sometimes |

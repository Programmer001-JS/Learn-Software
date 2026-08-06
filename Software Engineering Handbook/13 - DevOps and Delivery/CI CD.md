# CI CD

> **In one line —** integrating everyone's work continuously so merges stay small, and automating release so deploying becomes boring; the second only works because of the first.

| | |
|---|---|
| **Category** | Overview note *(hub for this sub-section)* |
| **Architectural Layer** | Delivery |
| **Sub-topics** | [GitHub Actions](GitHub%20Actions.md) · [Deployment Strategies](Deployment%20Strategies.md) · [Docker](Docker.md) · [Terraform](Terraform.md) |
| **Related notes** | [Git](Git.md) · [GitHub](GitHub.md) · [Software Development Lifecycle](../01%20-%20Foundation/05%20-%20Software%20Development%20Lifecycle.md) · [Monitoring and Alerting](../14%20-%20Scalability%20and%20Reliability/Monitoring%20and%20Alerting.md) |

---

## 1. Short Definition

*What is it?*

Three ideas that are routinely conflated:

```text
CONTINUOUS INTEGRATION    everyone merges to the trunk at least daily,
                          and every merge is automatically built and tested

CONTINUOUS DELIVERY       every commit that passes is RELEASABLE at any time;
                          the decision to ship is a human one

CONTINUOUS DEPLOYMENT     every commit that passes IS shipped automatically
```

---

## 2. Problem

*What engineering problem does it solve?*

```text
Six developers, six long-lived branches, integration "at the end"
    ↓
Three weeks of divergence
    ↓
Integration week: hundreds of conflicts, broken tests, nobody sure what changed
    ↓
A release is a rare, frightening event requiring a weekend and a rollback plan
    ↓
Because releases are frightening, they become rarer
    ↓
Because they are rarer, each one is bigger and more frightening
```

> [!IMPORTANT]
> **CI/CD breaks a feedback loop that gets worse on its own.** Large, rare releases are risky, so teams release less often, which makes each release larger and riskier. The way out is counter-intuitive: **deploy far more often, in far smaller pieces.** A ten-line change that goes out in an hour is easy to review, easy to reason about and trivial to roll back. That is the entire argument, and the tooling is secondary to it.

---

## 3. Architecture Position

```text
commit ──► CI ─────────────────────► artefact ──► CD ──────────► production
            │                          │                            │
            ├ lint, type check           immutable,                  ├ canary
            ├ unit tests                 tagged with                 ├ rollback
            ├ build                      the commit SHA              └ monitoring
            ├ security scan                                              feeds back
            └ integration tests
```

> [!TIP]
> **Build the artefact once and promote the same one through every environment.** Rebuilding per environment means staging and production run different bytes, which is how "it worked in staging" happens. Configuration comes from the environment; the artefact does not change.

---

## 4. Continuous integration is a practice, not a server

```text
HAVING A CI SERVER                DOING CONTINUOUS INTEGRATION
a pipeline exists                 everyone merges to trunk daily
branches live for weeks           branches live for hours
the build is often red            a red build is fixed before anything else
tests are sometimes skipped       the trunk is always releasable
```

> [!CAUTION]
> **Most teams that say they "do CI" have a build server and long-lived branches, which is not CI.** The word doing the work is *continuous*: if a branch lives for a week, integration is not continuous no matter how many pipelines run on it. Similarly, a build that is red for two days has stopped being a signal — a broken trunk should be the highest-priority item for whoever broke it, ahead of feature work.

---

## 5. The pipeline, in the right order

```text
FAST FEEDBACK FIRST — fail in seconds, not minutes
    1. lint, format, type check          seconds
    2. unit tests                        under a couple of minutes
    3. build the artefact                 minutes
    4. dependency + secret scanning       fast, run in parallel
    5. integration tests                  minutes
    6. deploy to staging automatically
    7. smoke / end-to-end tests           the slow suite
    8. deploy to production               automatic, or one approval
```

> [!TIP]
> **Order stages by how quickly they can prove you wrong, and parallelise anything independent.** A pipeline that spends eight minutes building before discovering a formatting error is wasting the most valuable resource in delivery, which is the developer's attention span. Everything under ten minutes stays in flow; beyond twenty, people context-switch and reviews go stale.

---

## 6. Flaky tests will destroy this if you let them

```text
One test fails 3% of the time
    ↓
Developers learn to re-run the pipeline
    ↓
Re-running becomes reflexive
    ↓
A real failure is re-run away without being read
    ↓
The suite no longer means anything
```

> [!CAUTION]
> **A flaky test is worse than no test, because it destroys trust in every other test.** Treat flakiness as a defect with an owner: quarantine the test immediately so it stops blocking, raise a ticket, and fix or delete it within a defined window. Teams that tolerate a handful of flaky tests reliably end up with a suite nobody believes and a habit of merging on a re-run.

---

## 7. Deployment must be reversible

```text
BEFORE automating deployment, automate:
    a health check that actually reflects readiness
    a rollback that takes ONE action
    monitoring that would tell you it went wrong
    a database migration strategy that is backward compatible
```

```text
Database migrations are the asymmetry:
    code rolls back in seconds
    a dropped column does not
    → expand, migrate, contract — never destructive in one release
```

> [!IMPORTANT]
> **Deployment speed is worth nothing without rollback speed.** The point of frequent small releases is that a mistake is cheap — and it is only cheap if reverting is a button rather than an incident. Practise the rollback deliberately, because the first time you use it should not be during an outage. See [Deployment Strategies](Deployment%20Strategies.md).

---

## 8. Feature flags decouple deploy from release

```text
DEPLOY   the code is in production        (a technical event)
RELEASE  users can see the behaviour      (a business decision)
    ↓
With flags these become independent:
    merge incomplete work safely behind a flag
    enable for 1% of users, then 50%, then everyone
    turn it off in seconds without a deployment
```

> [!TIP]
> **Feature flags are what make trunk-based development practical for large features.** Without them, "merge daily" and "do not ship half-finished work" conflict, and teams resolve the conflict with long branches. With them, incomplete work lives on the trunk, invisible. The discipline required is removing stale flags — an old flag is a hidden code path nobody tests.

---

## 9. Real World Example

- **A typical web team** — pull request runs tests, merge to main deploys to staging automatically, production behind one approval; several releases a day.
- **High-performing organisations** deploying dozens of times a day, entirely automatically, with canary analysis gating the rollout.
- **Mobile applications**, where store review forces continuous delivery rather than deployment.
- **Infrastructure pipelines** — a Terraform plan posted for review, applied on merge; see [Terraform](Terraform.md).
- **Machine learning delivery**, where the artefact includes a model version and data lineage; see [AI Pipelines](../11%20-%20AI%20Engineering/AI%20Pipelines.md).

---

## 10. Communication and Dependencies

- **Version control with protected branches**; see [Git](Git.md), [GitHub](GitHub.md)
- **A test suite that is trusted** — the load-bearing dependency
- **An artefact registry** — container images or packages, immutable and tagged by SHA
- **Credentials for deployment**, ideally OIDC rather than stored keys
- **Monitoring**, so an automated deployment can be judged; see [Monitoring and Alerting](../14%20-%20Scalability%20and%20Reliability/Monitoring%20and%20Alerting.md)
- **A rollback mechanism**, tested before it is needed

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Every project benefits from CI from the first week — even a solo one, because tests that run automatically are tests that keep working. Continuous *deployment* is worth it once you have real monitoring and a real rollback path.

> [!CAUTION]
> - **Do not automate deployment before you can observe and revert it** — you have automated the delivery of outages
> - **Do not deploy continuously without backward-compatible migrations**
> - **Do not gate merges on a suite that is flaky** — fix the suite first
> - **Do not run a twenty-minute pipeline on every commit** if a two-minute subset would catch most failures
> - **Do not use CI as a substitute for local checks** — pre-commit hooks catch trivia faster and cheaper
> - **Do not put manual approval on every stage** and call it delivery; that is a change advisory board with a nicer interface

---

## 12. Advantages and Disadvantages

**Advantages**
- Small changes, so defects are found near their cause
- Releases become routine instead of events
- Rollback is cheap, which makes risk-taking safe
- Every artefact is traceable to a commit
- Manual, error-prone release steps disappear
- Reviews stay fast because diffs stay small
- Measurable improvements in lead time and change failure rate

**Disadvantages**
- **Real investment in tests, and in keeping them trustworthy**
- Pipeline maintenance becomes ongoing work
- Flaky tests actively erode the value
- Database migrations require discipline that is easy to skip
- Feature flags accumulate into hidden complexity
- Compute cost for pipelines at scale
- The cultural change is harder than the tooling

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Pipeline duration** | Under 10 minutes keeps developers in flow; over 20 breaks it |
| **Caching** | Dependency and layer caches often halve build time |
| **Parallelism** | Split test suites across runners; the biggest available win |
| **Shallow clone** | Removes minutes on repositories with long histories |
| **Container builds** | Layer ordering matters more than any other Dockerfile choice |
| **Test selection** | Running only what the change affects, at large scale |

> [!TIP]
> **Measure your pipeline like production: know its p95 duration and where the time goes.** Most slow pipelines are slow for two or three findable reasons — an uncached dependency install, a serial test suite, and a full clone. Fixing those is usually a day's work and pays back every commit thereafter.

---

## 14. Security Considerations

> [!CAUTION]
> **Your CI system can deploy to production, which makes it one of the most valuable targets you own.** It holds credentials, runs third-party code, and its output is trusted by your infrastructure. A compromised pipeline is worse than a compromised server: it can ship a backdoor through every control you have, signed off as a normal release. Treat pipeline configuration with the same care as production access.

- **No long-lived cloud keys** — use OIDC federation; see [IAM](../12%20-%20Cloud%20Architecture/IAM.md)
- **Pin third-party actions and images to digests**, not mutable tags
- **Least privilege per job**, and no secrets in jobs that build untrusted code
- **Never expose secrets to pull requests from forks**
- **Scan dependencies and images in the pipeline**, and fail on critical findings
- **Sign artefacts** and verify signatures at deploy time (Sigstore, cosign)
- **Protect the pipeline definition** — changes to it need review like any other code
- **Ephemeral runners** — a self-hosted runner reused across jobs can leak state between them
- **Audit deployment events**, so you know what shipped and who approved it

---

## 15. Mental Model

> [!NOTE]
> **CI/CD is a factory production line with quality gates, replacing hand-assembly.**
>
> Every part enters the same line, passes the same inspections in the same order, and comes out as an identical, labelled unit. Nobody assembles a special one by hand for a customer. The gates are placed so the cheapest inspections come first, and the line stops when a defect is found — because a line that keeps running while producing defects is just making the problem larger. The goal is not speed for its own sake; it is that every unit is the same and every unit is checked.

---

## 16. Mini Architecture Diagram

```text
   developer commit
          │
   ┌──────▼─────────── PULL REQUEST PIPELINE ────────────────┐
   │  lint · types (30 s)                                     │
   │  unit tests (2 min)      ─── parallel ───                │
   │  dependency + secret scan                                 │
   │  build container image                                    │
   │  integration tests (5 min)                                │
   └──────┬───────────────────────────────────────────────────┘
          │ all green + review
          ▼
   protected main
          │
   ┌──────▼─────────── DELIVERY PIPELINE ────────────────────┐
   │  build ONCE → image:sha256 → registry (immutable)        │
   │        │                                                  │
   │        ├─► staging (auto)  → smoke tests                  │
   │        │                                                  │
   │        └─► production  ← approval or fully automatic      │
   │              canary 5% → metrics healthy? → 100%          │
   │              unhealthy? → automatic rollback              │
   └──────┬───────────────────────────────────────────────────┘
          │
   monitoring + error rate + p99  ──feeds back──► the next decision
```

---

## 17. Complete Request Flow

```text
Developer pushes a 60-line change
    ↓
Pipeline: lint and type check fail in 25 s on a missing return type — fixed
    ↓
Re-run: unit tests, scanning and build run in parallel; green in 4 minutes
    ↓
Review is quick, because the diff is small
    ↓
Merged to main; the artefact is built ONCE and tagged with the commit SHA
    ↓
Deployed automatically to staging; smoke tests pass
    ↓
Production deployment starts: canary at 5% of traffic
    ↓
Error rate and p99 compared against the previous version for 10 minutes
    ↓
Healthy → rolled forward to 100%
    ↓
The new feature is deployed but DISABLED behind a flag
    ↓
Product enables it for 1% of users, then 50%, then everyone — no deployment involved
    ↓
─────────────── it goes wrong ───────────────
Canary error rate triples within 2 minutes
    ↓
Automatic rollback to the previous image; traffic normal within 90 seconds
    ↓
Blast radius: 5% of traffic for 2 minutes, and no human was paged
    ↓
─────────────── and the migration ───────────────
The release needed a column removed
    ↓
Release 1: stop writing to it (code only, reversible)
Release 2: backfill and stop reading it
Release 3: drop the column — days later, once rollback is no longer plausible
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Integrate daily in small pieces, keep the pipeline under ten minutes, build the artefact once and promote it, never automate deployment before rollback and monitoring exist, and treat a flaky test as a defect rather than an inconvenience.

---

## 19. Common Mistakes

- **Calling it CI while running long-lived branches** — the practice is the merging, not the server
- **Tolerating flaky tests**, which trains everyone to re-run instead of read
- **A red trunk left broken** for days
- **Rebuilding the artefact per environment**, so staging and production differ
- **Automating deployment without rollback or monitoring**
- **Destructive database migrations** in the same release as the code change
- **A twenty-minute pipeline** on every commit, so people batch their work
- **Long-lived cloud credentials** in CI instead of OIDC
- **Secrets exposed to fork pull requests**
- **Manual approval at every stage**, which is the old process wearing new tooling
- **Feature flags never removed**, becoming untested hidden branches
- **No ownership of the pipeline**, so it decays until it is nobody's problem and everybody's blocker

---

## 20. Open Source Technologies

- **GitHub Actions**, **GitLab CI**, **Jenkins**, **Woodpecker**, **Drone** — pipeline engines
- **Argo CD**, **Flux** — GitOps delivery for Kubernetes
- **Spinnaker**, **Argo Rollouts** — canary and progressive delivery with automated analysis
- **Docker / BuildKit** — reproducible builds; layer caching is the main lever
- **Trivy**, **Grype**, **Checkov** — scan images, dependencies and infrastructure code
- **cosign / Sigstore** — sign artefacts and verify them at deploy time
- **Unleash**, **Flagsmith**, **OpenFeature** — self-hosted feature flags
- **pre-commit** — move fast checks left, off the pipeline entirely
- **act**, **dagger** — run pipelines locally

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Time your pipeline and write down where the minutes go. Fix the largest item.
- [ ] Count how long your branches live, in hours. That number is your real CI maturity.
- [ ] List your flaky tests. Quarantine them today and assign owners.
- [ ] Perform a rollback deliberately, in staging, and time it.
- [ ] Check whether your artefact is built once or rebuilt per environment.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
commit → fast checks → tests → build once → staging → canary → production
                                                        └── rollback + monitoring
```

## 2. Request Flow

```text
Input       a small change merged to the trunk daily
    ↓
Processing  cheap checks first, then tests, then one immutable artefact promoted forward
    ↓
Output      a release that is routine, observable and reversible
```

## 3. Real-World Usage

The research on this is unusually consistent: the organisations that deploy most frequently also have the *lowest* change failure rate, because small changes fail less and recover faster. The practices that produce it are unglamorous — short branches, a trusted test suite, one artefact promoted through environments, and a rollback that someone has actually used.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Continuous integration of small changes, plus automated, reversible release |
| **Why does it exist?** | Because rare large releases are risky, which makes releases rarer still |
| **Where does it belong?** | Between version control and production, gating both |
| **When should I use it?** | CI from day one; automated deployment once monitoring and rollback exist |

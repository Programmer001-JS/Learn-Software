# Terraform

> **In one line —** infrastructure declared as code and reconciled against a state file; the state file is both the reason it works and the source of nearly every problem you will have with it.

| | |
|---|---|
| **Category** | Infrastructure as Code |
| **Architectural Layer** | Infrastructure |
| **Related notes** | [Ansible](Ansible.md) · [AWS Architecture](../12%20-%20Cloud%20Architecture/AWS%20Architecture.md) · [CI CD](CI%20CD.md) · [Cloud Fundamentals](../12%20-%20Cloud%20Architecture/Cloud%20Fundamentals.md) · [Kubernetes](Kubernetes.md) · [Git](Git.md) |

---

## 1. Short Definition

*What is it?*

Terraform describes infrastructure **declaratively** in HCL, compares that description with a recorded **state**, and produces a plan of the API calls needed to close the gap. One tool, many providers — AWS, GCP, Cloudflare, GitHub, Datadog.

> **A note on names:** Terraform's licence changed in 2023 and **OpenTofu** is the open-source fork. Everything in this note applies to both; where new projects want a permissive licence, OpenTofu is the drop-in choice.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Infrastructure created by clicking in a console
    ↓
Nobody knows what exists, or who created it, or why
Staging and production have drifted apart in ways nobody can list
Rebuilding after a disaster means archaeology
A change cannot be reviewed before it happens
    ↓
The environment is undocumented by construction
```

> [!IMPORTANT]
> **The plan is the feature.** Being able to see exactly what will change — *before* it changes — turns infrastructure work from an act of faith into a reviewable diff. That, more than reuse or automation, is why Terraform won: a pull request showing "this will destroy the database" is worth more than any amount of documentation.

---

## 3. Architecture Position

```text
    .tf files  (declared desired state, in Git)
        │
    terraform plan  ──reads──► STATE FILE ──describes──► real infrastructure
        │                       (remote, locked)                  ▲
        │  a reviewable diff                                      │
        ▼                                                          │
    terraform apply ──────── provider ──── cloud API ─────────────┘
        │
    state UPDATED to match reality
```

Three things must agree: **your code, the state file, and reality.** Every Terraform problem is one of those three disagreeing with another.

---

## 4. State: the thing to understand first

```text
STATE MAPS  your resource addresses  →  real resource IDs
    aws_instance.web  →  i-0abc123...

WITHOUT STATE
    Terraform cannot tell "create a new one" from "this already exists"

STATE MUST BE
    REMOTE      S3 + DynamoDB lock, or Terraform Cloud, or GCS
    LOCKED      or two concurrent applies corrupt it
    VERSIONED   so you can recover a bad state
    TREATED AS SECRET   ⚠ it contains plaintext values
```

> [!CAUTION]
> **The state file contains secrets in plaintext — database passwords, generated keys, sensitive outputs — regardless of how carefully you marked variables as sensitive.** It must never be committed to Git, and the bucket holding it must be encrypted with tight access control. Anyone who can read your state can read your credentials.

> [!IMPORTANT]
> **Local state on a laptop is the beginning of every Terraform horror story.** Two engineers, two state files, one environment. Set up a remote backend with locking on day one — it takes ten minutes and prevents a class of problem that is genuinely painful to unwind.

---

## 5. The core commands

```text
terraform init      download providers, configure the backend
terraform fmt       canonical formatting — run it in CI
terraform validate  syntax and type checking, no API calls
terraform plan      what WOULD change  ← read this, every time
terraform apply     do it
terraform destroy   remove everything in this state  ⚠
terraform state ...  surgery: list, mv, rm, import
```

```text
READING A PLAN
    +  create
    -  destroy
    ~  update in place
-/+   DESTROY AND RECREATE   ← the one to look for
```

> [!CAUTION]
> **`-/+` means the resource will be destroyed and rebuilt, and for a database or a load balancer that is an outage.** Terraform is not being reckless; some attributes cannot be changed in place, so the provider replaces the resource. Read every plan for that symbol, and use `prevent_destroy` lifecycle rules on anything whose replacement would be catastrophic.

---

## 6. Structuring a project

```text
ONE STATE PER ENVIRONMENT AND PER BLAST RADIUS
    infra/
      modules/                 reusable, versioned, no hardcoded environments
        network/
        service/
      envs/
        staging/               its own state
        prod/                  its own state
      bootstrap/               the state bucket itself
```

```text
WHY SEPARATE STATES
    a mistake in staging cannot touch production
    plans stay fast (one state with 2,000 resources is painful)
    blast radius of `destroy` is bounded
```

> [!TIP]
> **Split state by rate of change as well as by environment.** Networking changes rarely; applications change daily. Keeping them in one state means every application deployment plans the whole VPC, which is slow and needlessly risky. A small number of states with clear boundaries — network, data, application — is far easier to live with than one enormous one, and far easier than fifty tiny ones.

> [!CAUTION]
> **Avoid Terraform workspaces for environment separation.** They share one backend key namespace and one configuration, which makes it easy to apply staging's plan to production. Separate directories with separate backends make the boundary explicit and visible in the file path.

---

## 7. Modules, without over-engineering

```hcl
module "api" {
  source = "git::https://github.com/org/tf-modules.git//service?ref=v1.4.0"

  name          = "api"
  environment   = var.environment
  instance_type = "t4g.small"
  desired_count = 3
}
```

```text
GOOD MODULE          one coherent thing, few required inputs, clear outputs
BAD MODULE           a wrapper around one resource that adds nothing
WORSE MODULE         forty variables, conditional everything, unreadable
```

> [!TIP]
> **Pin module sources to a version or tag, and do not abstract before the second use.** A module written for a single caller is speculation; the third caller is what teaches you the right interface. Meanwhile, an unpinned module source means your infrastructure can change because someone else committed to a repository.

---

## 8. Terraform in CI

```text
PULL REQUEST
    terraform fmt -check · validate · plan
    → post the plan as a comment; review it like code
    → scan with Checkov or tfsec for insecure configuration

MERGE TO MAIN
    terraform apply on the SAVED plan file
    → applying a re-run plan can apply something you never reviewed

CREDENTIALS
    OIDC to a cloud role, not stored keys — see IAM
```

> [!IMPORTANT]
> **Apply the plan file you reviewed, not a fresh plan.** `terraform plan -out=tfplan` then `terraform apply tfplan` guarantees that what was approved is what happens. Re-planning at apply time reopens the window for a change nobody looked at — including someone else's concurrent modification.

---

## 9. Real World Example

- **A whole AWS account from scratch** — VPC, subnets, EKS or ECS, RDS, IAM roles, monitoring.
- **Multi-environment platforms**, where staging and production come from the same modules with different inputs.
- **Disaster recovery** — the ability to rebuild an environment in a new region from code; see [Disaster Recovery](../14%20-%20Scalability%20and%20Reliability/Disaster%20Recovery.md).
- **Non-cloud providers** — GitHub repositories and branch protection, Cloudflare DNS, Datadog monitors, Okta.
- **Ephemeral review environments**, created per pull request and destroyed on merge.
- **Compliance evidence** — the Git history is the change record an auditor asks for.

---

## 10. Communication and Dependencies

- **A remote backend with locking** — non-negotiable for a team
- **Cloud credentials**, ideally via OIDC in CI; see [IAM](../12%20-%20Cloud%20Architecture/IAM.md)
- **Provider version constraints and a committed lock file**, or builds are not reproducible
- **Git**, because the history is the audit trail; see [Git](Git.md)
- **A configuration tool for inside the machine**, if you use instances; see [Ansible](Ansible.md)
- **Policy scanning in CI** — Checkov, tfsec, or OPA

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Terraform for anything with a provider and a lifecycle: networks, clusters, databases, DNS, IAM, SaaS configuration. If a resource is created once and lives for years, it belongs in code.

> [!CAUTION]
> - **Not for configuring the inside of a server** — that is Ansible or a container image
> - **Not for application deployment** — it can do it, but rollouts, canaries and health gating are not its strengths; see [Deployment Strategies](Deployment%20Strategies.md)
> - **Not for anything genuinely ephemeral** created and destroyed many times a day by an application
> - **Not with local state** in a team
> - **Not as a way to learn a cloud** — understand what the resource is before you declare it
> - **Not one giant state** for an entire organisation

---

## 12. Advantages and Disadvantages

**Advantages**
- The plan: changes are visible and reviewable before they happen
- One language and workflow across many providers
- Reproducible environments, and a real path to rebuilding after a disaster
- Git history becomes the infrastructure change record
- Modules make patterns reusable and consistent
- Drift is detectable — a plan on unchanged code should be empty
- The de facto standard, so knowledge and modules are widely available

**Disadvantages**
- **State is fragile, sensitive, and the source of most incidents**
- HCL is limited: loops, conditionals and dynamic structures get awkward fast
- Provider bugs and coverage gaps are real and occasionally blocking
- Slow plans on large states
- Refactoring resource addresses requires `state mv` or `moved` blocks
- Destructive replacements are easy to miss in a long plan
- Importing existing infrastructure is tedious
- Cross-state references (`remote_state`) create hidden coupling

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Plan time** | Grows with resource count; refresh calls every resource's API |
| **`-refresh=false`** | Much faster plans, at the risk of missing drift |
| **`-target`** | An escape hatch for slow plans, and a habit worth avoiding |
| **Parallelism** | Defaults to 10 concurrent operations; tuneable |
| **Provider API rate limits** | Large applies get throttled; retries are built in |
| **State size** | Split large states rather than tolerating five-minute plans |

> [!TIP]
> **A slow plan is a structural signal, not a tooling problem.** If planning takes five minutes, the state is too large, and the fix is splitting it along a boundary that matters — not reaching for `-target`, which produces partial applies and state that no longer matches any reviewed plan.

---

## 14. Security Considerations

> [!CAUTION]
> **Terraform credentials are usually the most powerful in your organisation, because creating infrastructure implies the ability to destroy it.** A compromised CI pipeline with an admin Terraform role can delete an entire environment. Scope the role to what the configuration actually manages, require review on the configuration itself, and keep the state bucket in an account the pipeline cannot administer.

- **Remote state encrypted, versioned, access-controlled**, and never in Git
- **Treat state as secret** — it contains plaintext values
- **OIDC rather than long-lived cloud keys** in CI
- **`prevent_destroy`** on databases, state buckets and anything irreplaceable
- **Never store real secrets in `.tf` files or `.tfvars`** — reference a secret manager and accept that the value still lands in state
- **Scan configuration in CI** — Checkov and tfsec catch public buckets and open security groups before they exist
- **Pin provider versions and commit the lock file** — a provider is code that runs with your credentials
- **Separate roles per environment**, so a staging apply cannot reach production

---

## 15. Mental Model

> [!NOTE]
> **Terraform is a building's blueprint, plus an inventory of what was actually built.**
>
> The blueprint says what should exist. The inventory records what was built and which brick is which. Terraform's job is to compare the two, tell you the difference, and then do the work. If someone moves a wall by hand, the inventory is now wrong and the next comparison produces surprises — that is drift. And if you lose the inventory, the tool cannot tell an existing building from an empty plot, which is why the state file matters as much as the blueprint.

---

## 16. Mini Architecture Diagram

```text
   infra/
     modules/network · modules/service        (versioned, pinned)
     envs/staging  →  backend: s3://tf-state/staging.tfstate
     envs/prod     →  backend: s3://tf-state/prod.tfstate
                                  │
   ┌──────────────────────────────┼───────────────────────────────┐
   │  PULL REQUEST                │                                │
   │    fmt · validate            │                                │
   │    plan -out=tfplan ─────────┤ reads state (read-only role)   │
   │    checkov / tfsec           │                                │
   │    plan posted as a comment  │                                │
   └──────────────┬───────────────┘                                │
                  │ review + approve                               │
   ┌──────────────▼────────────────────────────────────────────────┤
   │  MERGE TO MAIN                                                 │
   │    apply tfplan   ← the SAME plan that was reviewed            │
   │    OIDC → apply role (scoped, per environment)                 │
   └──────────────┬────────────────────────────────────────────────┘
                  ▼
      cloud APIs → real resources → state updated
                  │
      S3 state bucket: encrypted · versioned · DynamoDB lock
      (in an account the pipeline cannot administer)
```

---

## 17. Complete Request Flow

```text
Engineer edits envs/prod/main.tf to add a read replica
    ↓
Opens a pull request; CI runs fmt, validate, then plan with a read-only role
    ↓
Plan output posted as a comment:
    + aws_db_instance.replica
    ~ aws_security_group.db  (one ingress rule)
    ↓
Checkov flags nothing; a reviewer reads the plan and approves
    ↓
Merge → CI assumes the prod apply role via OIDC
    ↓
DynamoDB lock acquired — a concurrent apply would now wait, not corrupt
    ↓
`terraform apply tfplan` — the exact reviewed plan, no re-planning
    ↓
Resources created; state updated in S3 (versioned, encrypted)
    ↓
Lock released
    ↓
─────────────── drift ───────────────
Someone changes a security group by hand during an incident
    ↓
The next plan shows an unexpected `~` — drift is visible
    ↓
Either codify the change or let Terraform revert it, deliberately
    ↓
─────────────── a dangerous plan ───────────────
An engineer changes a database's engine version attribute
    ↓
Plan shows -/+ aws_db_instance.main  (DESTROY AND RECREATE)
    ↓
Caught in review; `prevent_destroy` would have blocked the apply anyway
    ↓
Done as a managed upgrade instead
    ↓
─────────────── a refactor ───────────────
A resource is renamed in code
    ↓
Terraform sees a destroy plus a create, which is wrong
    ↓
A `moved` block (or `state mv`) tells it the resource is the same one
    ↓
Plan becomes empty; nothing is rebuilt
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Remote locked state from day one, one state per environment, read every plan for `-/+` before applying, apply the exact plan you reviewed, and treat the state file as both critical and secret.

---

## 19. Common Mistakes

- **Local state**, or state committed to Git
- **No state locking**, so two applies corrupt each other
- **Secrets in `.tf` or `.tfvars`**, and forgetting they land in state regardless
- **One giant state** for everything, making plans slow and `destroy` terrifying
- **Workspaces used for environments**, blurring the production boundary
- **Applying a fresh plan** instead of the reviewed plan file
- **Skimming the plan** and missing a `-/+` replacement
- **No `prevent_destroy`** on databases and state buckets
- **Unpinned module sources and providers**, so infrastructure changes without a commit
- **`-target` as a habit**, producing partial state
- **Manual console changes**, creating drift nobody codifies
- **Using it to configure inside servers**, where Ansible or an image belongs
- **Abstracting into modules before the second use case exists**

---

## 20. Open Source Technologies

- **OpenTofu** — the open-source fork; drop-in compatible
- **Terragrunt** — DRY backends and multi-state orchestration; useful, adds a layer
- **Pulumi**, **AWS CDK** — the same job in a general-purpose language
- **tflint**, **tfsec**, **Checkov**, **Terrascan** — linting and security scanning
- **Infracost** — the cost impact of a change, in the pull request
- **Atlantis** — pull-request-driven plan and apply, self-hosted
- **terraform-docs** — generate module documentation from the code
- **tfstate viewers and `terraform state` subcommands** — for the surgery you will eventually need
- **Terratest**, **terraform test** — actually test modules

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Confirm your state is remote, encrypted, versioned and locked.
- [ ] Run a plan on unchanged code. Anything other than "no changes" is drift — investigate it.
- [ ] Add `prevent_destroy` to your database and your state bucket.
- [ ] Add a plan-on-pull-request step with the plan posted as a comment.
- [ ] Time your plan. If it exceeds two minutes, decide where you would split the state.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
.tf in Git → plan (vs remote locked state) → review → apply the reviewed plan → cloud
```

## 2. Request Flow

```text
Input       a declarative description of desired infrastructure
    ↓
Processing  compare against recorded state and reality, produce a reviewable diff
    ↓
Output      API calls that close the gap, and an updated state file
```

## 3. Real-World Usage

Terraform became the standard because the plan made infrastructure reviewable, and it stayed the standard because one workflow covers every provider a company uses. The teams that operate it calmly all did the same unglamorous things early: remote locked state, separate state per environment, plans reviewed in pull requests, and OIDC instead of stored keys.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Declarative infrastructure reconciled against a recorded state file |
| **Why does it exist?** | Because clicked-together infrastructure is undocumented and unrepeatable |
| **Where does it belong?** | Around long-lived resources — networks, clusters, databases, DNS, IAM |
| **When should I use it?** | For anything created once and expected to last; never with local state in a team |

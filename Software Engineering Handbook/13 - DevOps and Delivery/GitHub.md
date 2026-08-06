# GitHub

> **In one line —** Git hosting that won by making the pull request the unit of collaboration, and has since become the place where code is reviewed, built, scanned and released.

| | |
|---|---|
| **Category** | Code Hosting and Collaboration Platform |
| **Architectural Layer** | Development |
| **Related notes** | [Git](Git.md) · [GitHub Actions](GitHub%20Actions.md) · [CI CD](CI%20CD.md) · [Code Review Checklist](../16%20-%20Templates/Code%20Review%20Checklist.md) · [IAM](../12%20-%20Cloud%20Architecture/IAM.md) |

---

## 1. Short Definition

*What is it?*

GitHub hosts Git repositories and adds the layer Git deliberately does not have: **pull requests, code review, issues, permissions, automation and releases.** Git manages history; GitHub manages the human process around it.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Git gives you distributed history
    ↓
It does NOT give you:
    a shared place everyone agrees is canonical
    a way to propose and discuss a change before it lands
    a record of WHY a change was accepted
    access control
    automation triggered by a change
    ↓
Git solved versioning. Collaboration was still email and patches.
```

> [!IMPORTANT]
> **The pull request is GitHub's actual contribution, and it changed how software is written.** It attached discussion, review and automated checks to a specific proposed change, before that change reached the main branch. Everything else — Actions, protected branches, required reviews, code scanning — is machinery hanging off that one idea, and it is why "open a PR" is now a universal instruction across the industry.

---

## 3. Architecture Position

```text
    Developer's local repo  (Git)
              │ push a branch
              ▼
    ┌──────── GITHUB ─────────────────────────────┐
    │  repository + branches + tags               │
    │        │                                     │
    │  PULL REQUEST  ← the coordination point      │
    │        ├── review comments                   │
    │        ├── required status checks (Actions)  │
    │        ├── code scanning, secret scanning     │
    │        └── branch protection rules            │
    │        │ merge                                │
    │        ▼                                      │
    │  main → release / tag → package registry     │
    └──────────┬───────────────────────────────────┘
               │ OIDC (no stored keys)
               ▼
        cloud environment — see IAM
```

---

## 4. The pull request, used well

```text
A GOOD PULL REQUEST
    small — under a few hundred changed lines
    one purpose, stated in the description
    says WHY, not just what (the diff already says what)
    tests included, CI green before review is requested
    a linked issue for context

A BAD PULL REQUEST
    2,000 lines across four unrelated concerns
    "misc fixes"
    opened with failing checks
    review requested from six people, so nobody feels responsible
```

> [!IMPORTANT]
> **Review quality collapses with size, and the drop-off is sharp.** Reviewers find real defects in a 50-line change and skim a 500-line one, approving it on trust. If a change cannot be made small, split it into a stack of sequential pull requests. Nothing else in this note improves engineering outcomes as reliably as smaller diffs.

> [!TIP]
> **Ask for review from one named person plus a team, not from five individuals.** Diffused responsibility produces slow, shallow reviews — everybody assumes someone else is reading carefully.

---

## 5. Branch protection: the settings that matter

```text
ON MAIN, ENABLE
    require a pull request before merging
    require status checks to pass (name the specific checks)
    require at least one approving review
    require branches to be up to date before merging
    dismiss stale approvals when new commits are pushed
    block force pushes and deletion
    INCLUDE ADMINISTRATORS      ← the one people skip
```

> [!CAUTION]
> **Protection that exempts administrators protects nothing under pressure.** The moment that matters is a Friday evening incident when someone with admin rights bypasses review "just this once" — which is precisely when a second pair of eyes had the most value. Include administrators, and use an explicit break-glass procedure that is logged instead.

---

## 6. Beyond hosting

| Feature | What it is for | Worth it? |
|---|---|---|
| **Actions** | CI/CD in the same place as the code | Yes; see [GitHub Actions](GitHub%20Actions.md) |
| **Dependabot** | Dependency updates and vulnerability alerts | Yes — enable it, then actually triage |
| **Secret scanning** | Finds committed credentials, with push protection | Yes, unconditionally |
| **CodeQL** | Static analysis for security defects | Valuable, noisy at first |
| **Environments** | Deployment approvals and scoped secrets | Yes, for production |
| **CODEOWNERS** | Automatic review routing by path | Yes, on any multi-team repository |
| **Packages / GHCR** | Container and package registry | Convenient, competent |
| **Projects / Issues** | Planning | Fine; teams often use something else |
| **Codespaces** | Cloud development environments | Useful for onboarding; costs real money |

> [!TIP]
> **Turn on secret scanning with push protection today.** It rejects a push that contains a recognisable credential, which prevents the whole rotate-and-scrub exercise rather than reporting it afterwards. Combined with a Dependabot triage habit, it is the highest-value security work available for the least effort.

---

## 7. Permissions and access

```text
ORGANISATION
    └── TEAMS               grant access by team, never person by person
          └── REPOSITORIES  read / triage / write / maintain / admin

MACHINE ACCESS, in order of preference
    1. OIDC to a cloud role          → no stored secret at all
    2. GitHub App with fine-grained  → scoped, auditable
    3. Fine-grained PAT              → scoped, expiring
    4. Classic PAT                   → broad and long-lived; avoid
    5. A shared account's password   → never
```

> [!IMPORTANT]
> **OIDC removes the largest class of CI credential risk.** Instead of storing cloud keys in repository secrets, the workflow presents a short-lived token that your cloud trusts for a specific repository and branch. There is nothing to leak, nothing to rotate, and the trust policy states exactly which workflow may deploy. See [IAM](../12%20-%20Cloud%20Architecture/IAM.md).

---

## 8. Repository hygiene

```text
EVERY REPOSITORY SHOULD HAVE
    README            what it is, how to run it, how to test it
    .gitignore        correct from the first commit
    CODEOWNERS        who reviews what
    a PR template     the questions a reviewer always asks anyway
    branch protection on main
    CI that runs on every pull request
    SECURITY.md       how to report a vulnerability
    a licence         especially if public
```

> [!TIP]
> **A pull request template is a cheap, permanent improvement to review quality.** Three prompts — what changed, why, and how it was verified — convert vague descriptions into useful ones without anybody having to nag. The same applies to an issue template.

---

## 9. Real World Example

- **Effectively all open source** — the network effect is the product.
- **Internal engineering at most companies**, with Actions as the delivery pipeline.
- **GitOps** — a repository holds the desired state of a Kubernetes cluster; see [Kubernetes](Kubernetes.md).
- **Infrastructure review** — Terraform plans posted as pull request comments; see [Terraform](Terraform.md).
- **Release automation** — a tag produces artefacts, changelogs and container images.
- **This handbook**, and documentation generally.

---

## 10. Communication and Dependencies

- **Git** — GitHub is a host, not a replacement; see [Git](Git.md)
- **Actions runners** — hosted or self-hosted; see [GitHub Actions](GitHub%20Actions.md)
- **A cloud trust relationship** for OIDC-based deployment
- **A container or package registry**, GHCR or elsewhere
- **Status checks** that are actually required, not merely present
- **An identity provider** — SSO with enforced MFA, at any organisational size

---

## 11. When To Use / When NOT To Use

> [!TIP]
> GitHub is the default choice: the largest ecosystem, the best integrations, and the platform most engineers already know. For open source there is no realistic alternative because the audience is already there.

> [!CAUTION]
> - **Not if your code may not leave your premises** — self-host GitLab or Gitea instead
> - **Not as your artefact store for large binaries** — use object storage
> - **Not as a secrets manager** — repository secrets are for CI, not for application runtime
> - **Not as your only copy** — an organisation-level mistake or account compromise can remove access; mirror what matters
> - **Not with Actions for everything** by default if you already run a capable pipeline elsewhere

---

## 12. Advantages and Disadvantages

**Advantages**
- The pull request workflow, which the whole industry understands
- CI, code scanning, dependency management and hosting in one place
- Excellent third-party integration surface
- OIDC deployment without stored credentials
- Fine-grained branch protection and required checks
- Free and generous for public repositories
- The network effect for open source contribution

**Disadvantages**
- **Vendor concentration** — code, CI, registry and identity in one provider
- Outages block merging and deployment simultaneously
- Actions minutes and Codespaces cost grow quietly
- Permissions have several overlapping models (teams, apps, PATs, environments)
- Issues and Projects are weaker than dedicated tools
- Large monorepos strain the interface
- Default settings are permissive; protection is opt-in

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Clone from CI** | Use shallow clones; full history is rarely needed |
| **Large repositories** | Slow diffs and reviews; consider sparse checkout |
| **Actions queue time** | Hosted runners can queue; self-hosted removes that |
| **Required checks** | The slowest check sets your merge latency |
| **Large binaries** | Repository size grows permanently; use LFS or object storage |
| **API rate limits** | Real, and hit by automation that polls instead of using webhooks |

> [!TIP]
> **Your slowest required check is your delivery cadence.** If the pipeline takes forty minutes, changes batch up and reviews go stale. Parallelise, cache dependencies, and split the long end-to-end suite so a fast subset gates the merge — see [CI CD](CI%20CD.md).

---

## 14. Security Considerations

> [!CAUTION]
> **A third-party Action running in your repository executes arbitrary code with access to your secrets and, if misconfigured, your cloud.** Pin actions to a full commit SHA rather than a mutable tag, review what you adopt, and set the default workflow token to read-only. Supply chain attacks through popular actions and packages are now a routine attack path, not a theoretical one.

- **Enforce SSO with MFA** across the organisation
- **Secret scanning with push protection**, on every repository
- **OIDC rather than stored cloud credentials**
- **Pin third-party actions to a SHA**; prefer verified publishers
- **`permissions: read-all` as the default** in workflows, escalating per job
- **Protect main, including administrators**, and require signed commits where provenance matters
- **Beware `pull_request_target`** — it runs with secrets against untrusted code from forks
- **Audit organisation and repository access quarterly**; stale write access accumulates
- **Environments with required reviewers** for production deployments

---

## 15. Mental Model

> [!NOTE]
> **GitHub is the shared workshop around Git's toolbox.**
>
> Git is what you use at your own bench. GitHub is the room everyone works in: a noticeboard for proposed changes, a rule that nothing goes into the finished pile without a second person signing it off, an inspection machine that runs automatically, and a locked cupboard for the keys. The tools would still work without the workshop — but nobody would agree on what "done" meant.

---

## 16. Mini Architecture Diagram

```text
   developer ──push branch──► GITHUB
                                │
                    ┌───────────▼─────────────────────┐
                    │        PULL REQUEST              │
                    │  ┌────────────────────────────┐  │
                    │  │ required checks (Actions)   │  │
                    │  │  lint · test · build · scan │  │
                    │  └────────────────────────────┘  │
                    │  CODEOWNERS review requested     │
                    │  secret scanning · CodeQL        │
                    └───────────┬─────────────────────┘
                                │ merge (protected main)
                                ▼
                        main branch
                                │
                    ┌───────────▼───────────┐
                    │ release workflow      │
                    │ build image → GHCR    │
                    └───────────┬───────────┘
                                │ OIDC — no stored keys
                                ▼
                    environment: production
                    (required reviewer approval)
```

---

## 17. Complete Request Flow

```text
Developer pushes feature/login
    ↓
Opens a pull request; the template prompts what, why and how verified
    ↓
CODEOWNERS routes review to the team owning that path
    ↓
Actions runs lint, unit tests, build and CodeQL in parallel
    ↓
Secret scanning inspects the diff; push protection already blocked one earlier
    ↓
Dependabot notes the PR bumps a package with a known advisory — resolved
    ↓
One check fails → the author pushes a fix → stale approvals dismissed
    ↓
All required checks green, one approval → merge allowed
    ↓
Squash merged into protected main
    ↓
Release workflow builds a container image tagged with the commit SHA → GHCR
    ↓
Deployment to the production environment waits for a required reviewer
    ↓
Approved → workflow assumes a cloud role via OIDC (nothing stored)
    ↓
Deployed; the running image is traceable to a reviewed commit
    ↓
─────────────── a bad day ───────────────
An engineer with admin rights tries to push directly to main
    ↓
Blocked, because protection includes administrators
    ↓
They open a one-commit PR instead; CI catches a broken migration
    ↓
The rule that felt like friction prevented an outage
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Keep pull requests small, protect main including administrators, require the checks that matter, enable secret scanning with push protection, and deploy with OIDC so there are no cloud keys to leak.

---

## 19. Common Mistakes

- **Enormous pull requests** that get approved without real review
- **Branch protection that exempts administrators**
- **Status checks that exist but are not required**
- **Classic long-lived personal access tokens** instead of OIDC or GitHub Apps
- **Third-party actions pinned to a tag** rather than a commit SHA
- **A default write-scoped workflow token** on every job
- **`pull_request_target` used carelessly**, exposing secrets to fork code
- **Dependabot enabled and ignored**, producing hundreds of unread pull requests
- **No CODEOWNERS**, so review requests are guesswork
- **Secrets in the repository** rather than in secret storage
- **Assuming GitHub is a backup** of anything

---

## 20. Open Source Technologies

- **GitLab**, **Gitea**, **Forgejo** — self-hosted alternatives with the same workflow
- **gh** — the official CLI; scriptable and genuinely faster than the web interface
- **act** — run Actions workflows locally before pushing
- **pre-commit** — catch what CI would catch, before the commit exists
- **gitleaks**, **trufflehog** — secret scanning you control
- **Renovate** — dependency updates, more configurable than Dependabot
- **semantic-release** — automate versioning and changelogs from commit messages

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Check whether branch protection on main includes administrators. Fix it if not.
- [ ] Enable secret scanning with push protection on your main repository.
- [ ] Look at your last ten merged pull requests and record their sizes. Be honest about which were genuinely reviewed.
- [ ] Replace one stored cloud credential with an OIDC role.
- [ ] Add a CODEOWNERS file and a pull request template.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
local Git → branch → pull request (checks + review + scanning) → protected main
                                                              → release → OIDC → deploy
```

## 2. Request Flow

```text
Input       a proposed change on a branch
    ↓
Processing  automated checks, security scanning, human review, protection rules
    ↓
Output      a merge into main that is reviewed, tested and traceable to a deployment
```

## 3. Real-World Usage

GitHub's dominance rests on a single workflow that became the industry's shared vocabulary. The teams that get the most from it are not those using the most features — they are the ones whose pull requests are small, whose main branch genuinely cannot be bypassed, and whose CI has no stored cloud credentials to steal.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Git hosting plus the review, automation and permission layer around it |
| **Why does it exist?** | Because Git solved history but not collaboration |
| **Where does it belong?** | Between a developer's machine and the delivery pipeline |
| **When should I use it?** | By default — unless your code is not permitted to leave your premises |

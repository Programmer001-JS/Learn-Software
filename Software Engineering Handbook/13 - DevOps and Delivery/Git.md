# Git

> **In one line —** a content-addressed database of snapshots that happens to have a version control interface; everything confusing about it makes sense once you see the graph underneath.

| | |
|---|---|
| **Category** | Version Control System |
| **Architectural Layer** | Development |
| **Related notes** | [GitHub](GitHub.md) · [CI CD](CI%20CD.md) · [GitHub Actions](GitHub%20Actions.md) · [Deployment Strategies](Deployment%20Strategies.md) · [Code Review Checklist](../16%20-%20Templates/Code%20Review%20Checklist.md) |

---

## 1. Short Definition

*What is it?*

Git is a **distributed version control system**. Every clone contains the entire history, commits are immutable snapshots identified by the hash of their content, and branches are nothing more than a movable label pointing at one commit.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Multiple people changing the same files
    ↓
Whose version is current?
What did this line look like last month, and why?
How do we work on two features at once?
How do we know which code is in production?
    ↓
Without version control, the answer to all four is "ask someone"
```

Earlier systems answered this with a central server and file locking. Git's contribution was making the history **local and cheap**, so branching, committing and experimenting stopped requiring permission or a network.

> [!IMPORTANT]
> **Git's design decision that changed working practice was making branches free.** In older systems a branch was an expensive copy, so teams avoided them and integrated rarely. When a branch costs one 40-byte file, you branch per task — and pull requests, code review and CI on every change all become possible. Modern delivery practice is downstream of that one implementation detail.

---

## 3. Architecture Position

```text
WORKING DIRECTORY     the files you edit
        │  git add
        ▼
STAGING AREA (index)  what will go into the next commit
        │  git commit
        ▼
LOCAL REPOSITORY      the full object database and history
        │  git push / git fetch
        ▼
REMOTE REPOSITORY     a shared copy — GitHub, GitLab, a bare repo
        │
        ▼
CI/CD                 triggered by what arrives there
```

The staging area is the part beginners resent and experienced users rely on: it lets you commit **part** of your working changes, which is what makes small, coherent commits practical.

---

## 4. The object model — the thing worth actually understanding

```text
BLOB      file contents (no name, no path)
TREE      a directory: names → blobs and trees
COMMIT    a tree + parent commit(s) + author + message
TAG       a named pointer, usually to a release

Every object is identified by the SHA of its content.
Change one byte → a different hash → a different object.
```

```text
BRANCH = a 40-character file containing one commit hash
HEAD   = a pointer to the branch you are on
```

> [!IMPORTANT]
> **Commits are snapshots, not diffs.** Git stores complete trees and computes differences when you ask to see them. This is why checking out an old commit is instant, why history cannot be quietly altered (the hash would change), and why "rewriting history" always means **creating new commits and moving a label** — never editing the old ones. Once you see branches as labels on a graph, rebase, merge, reset and cherry-pick stop being magic incantations.

---

## 5. Merge versus rebase

```text
MERGE                              REBASE
creates a merge commit             replays your commits onto the new base
history shows what really happened  history is linear and easy to read
never rewrites existing commits    CREATES NEW COMMITS (new hashes)
safe on shared branches            dangerous on shared branches
```

```text
The workable rule:
    rebase your own unpushed work to tidy it
    merge (or squash) into the shared branch
    never rebase a branch someone else has based work on
```

> [!CAUTION]
> **`git push --force` on a shared branch is how teams lose work.** It replaces the remote's history with yours, discarding commits others have pushed. Use `--force-with-lease` instead — it refuses if the remote has moved since you last fetched, which is exactly the situation that causes the damage. Protect your main branch against force pushes at the platform level as well; see [GitHub](GitHub.md).

---

## 6. The commands that matter, and what they actually do

```text
git status                what is staged, unstaged, untracked
git add -p                stage HUNKS interactively — the habit that produces good commits
git commit                snapshot the index
git log --oneline --graph see the graph you are working on
git diff / --staged       before staging / after staging
git switch / restore      the modern, unambiguous replacement for checkout's two jobs
git rebase -i             reorder, squash, reword YOUR OWN commits
git stash                 park changes briefly
git bisect                binary search history for the commit that broke it
git reflog                where HEAD has been — the undo of last resort
```

> [!TIP]
> **`git reflog` recovers almost everything, and almost nobody learns it before they need it.** A bad reset, a lost commit after a rebase, a deleted branch — the commits still exist for weeks, and reflog shows every position HEAD held. Learn it on a calm afternoon rather than during a panic. Similarly, `git bisect` finds a regression across hundreds of commits in a few minutes, and most engineers debug by hand for hours instead.

---

## 7. Undoing things, without guessing

```text
"I want to change the last commit message"       git commit --amend
"I staged the wrong file"                        git restore --staged <file>
"Discard my local changes to a file"             git restore <file>
"Undo the last commit, KEEP the changes"         git reset --soft HEAD~1
"Undo the last commit, DISCARD the changes"      git reset --hard HEAD~1   ⚠
"Undo a commit that is already pushed"           git revert <sha>
"I destroyed something and I do not know what"   git reflog
```

> [!CAUTION]
> **`reset --hard` and `revert` are not alternatives — they belong to different situations.** Reset rewrites your local history and is fine for unpushed work. Revert creates a *new* commit undoing an old one, which is the only safe option once others have pulled it. Using reset on shared history is the single most common way to create an afternoon of confusion for a whole team.

---

## 8. Branching strategy, briefly

```text
TRUNK-BASED (recommended for most teams)
    short-lived branches, merged within a day or two
    main is always deployable, protected by CI
    incomplete work hidden behind feature flags

GIT FLOW
    develop, release, hotfix, feature branches
    designed for versioned software with scheduled releases
    heavy overhead for a continuously deployed web service
```

> [!TIP]
> **Branch lifetime matters far more than branch naming.** A branch open for three weeks produces a painful merge regardless of the strategy printed on the wall, because the divergence — not the workflow diagram — is what causes conflicts. If merges routinely hurt, the fix is smaller and shorter branches, not a more elaborate model.

---

## 9. Real World Example

- **Every modern software project**, effectively without exception.
- **The Linux kernel** — Git was written for it, which explains the emphasis on distributed work and patch review.
- **Infrastructure as code** — the Git history *is* the change record for your environment; see [Terraform](Terraform.md).
- **GitOps** — a repository is the declared desired state, and a controller reconciles the cluster to it; see [Kubernetes](Kubernetes.md).
- **Documentation and this handbook** — prose benefits from history and review as much as code does.

---

## 10. Communication and Dependencies

- **A remote** — GitHub, GitLab, or any host, including a bare repository over SSH
- **SSH keys or a token** for authentication; commit signing if provenance matters
- **`.gitignore`** — before the first commit, not after committing `node_modules`
- **CI**, which is triggered by pushes and pull requests; see [CI CD](CI%20CD.md)
- **Git LFS**, if binary assets are involved
- **Branch protection**, which is a platform feature rather than a Git one

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Git for anything textual whose history matters — code, configuration, infrastructure, documentation, SQL migrations, prompts. If a change to a file could ever need explaining, it belongs in version control.

> [!CAUTION]
> - **Not for large binary files** — every version is stored in full and clones grow forever; use LFS or object storage
> - **Not for secrets** — history is permanent, and a deleted secret is still in the repository; see [Secrets Management](../09%20-%20Security/Secrets%20Management.md)
> - **Not as a deployment mechanism** by pulling on a production server; build an artefact
> - **Not as a backup system** — a repository is a history, not a snapshot of your machine
> - **Not for generated files** — commit the source, build the output

---

## 12. Advantages and Disadvantages

**Advantages**
- Full history locally; almost every operation is fast and offline
- Branching and merging are cheap, which enables review and CI on every change
- Content addressing makes history tamper-evident
- Recovery from mistakes is nearly always possible via reflog
- The universal standard, so tooling and knowledge transfer everywhere
- Excellent at merging text

**Disadvantages**
- **A genuinely confusing command surface** with overlapping and historically inconsistent verbs
- Error messages describe internals rather than intent
- Poor with binary and very large files
- Submodules are widely disliked and rarely the right answer
- Rewriting history is powerful and easy to do damage with
- The mental model must be learned; the interface does not teach it

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Most operations** | Local and effectively instant |
| **Clone** | Proportional to full history; use `--depth 1` in CI |
| **Large binaries** | Every version stored in full — repository size grows without bound |
| **Many small files** | `status` slows on very large trees; enable fsmonitor |
| **Shallow and partial clone** | The standard fix for slow CI checkouts |
| **Monorepos** | Need sparse checkout and other deliberate tuning at scale |

> [!TIP]
> **In CI, clone shallowly.** A full clone of a long-lived repository can dominate a fast pipeline's runtime. `--depth 1` with a single branch is usually all a build needs — and if a step genuinely needs history, such as generating a changelog, fetch more only for that step.

---

## 14. Security Considerations

> [!CAUTION]
> **A secret committed to Git is compromised, and deleting it in a later commit does nothing.** It remains in the history, in every clone anyone made, and in any fork or CI cache. The only correct response is to **rotate the credential**, then optionally scrub the history. Treat "we removed it in the next commit" as a red flag in any review.

- **Secret scanning in CI and as a pre-commit hook** — gitleaks or trufflehog
- **`.gitignore` for `.env`, key files and credential caches**, from the first commit
- **Signed commits** where provenance matters — a commit's author field is trivially forgeable
- **Branch protection** requiring review and passing checks; enforce it on administrators too
- **Beware `git hooks` from untrusted repositories**, and be cautious with third-party actions in CI
- **Audit who has write access**, and prefer short-lived tokens over personal access tokens

---

## 15. Mental Model

> [!NOTE]
> **Git is a directed graph of immutable snapshots, and branches are sticky notes you move between them.**
>
> Commits are photographs already taken — nothing edits one. When you "rewrite history" you take new photographs and move the sticky note to the newest; the old ones sit in the drawer for a few weeks (that is reflog) before being swept up. Merge glues two lines together with a photograph that has two parents. Rebase re-photographs your work against a different backdrop. Every confusing Git situation becomes tractable by asking: *where is the note, and which photographs can I still reach?*

---

## 16. Mini Architecture Diagram

```text
   working dir ──add──► index ──commit──► local repo ──push──► remote
        ▲                                     │                  │
        └────────── restore / switch ──────────┘                  │
                                                                  ▼
                                                        CI: build, test, scan
                                                                  │
                                                                  ▼
                                                        deploy from the artefact

   THE GRAPH
        A ── B ── C ── D          ◄── main
                   \
                    E ── F        ◄── feature/login   (HEAD)

   branch = a file containing one hash
   HEAD   = which branch you are on
```

---

## 17. Complete Request Flow

```text
git switch -c feature/login          → a new label at the current commit
    ↓
edit files                            → working directory diverges
    ↓
git add -p                            → stage only the coherent hunks
    ↓
git commit                            → a new snapshot; the label moves
    ↓
git fetch && git rebase origin/main   → replay your commits on the latest main
    ↓                                   (new hashes — safe, nobody else has them)
git push --force-with-lease           → safe because the lease check protects others
    ↓
Pull request opened → CI runs on the merge result → review
    ↓
Squash merge into main                → one clean commit on the trunk
    ↓
CI on main builds an artefact tagged with the commit SHA
    ↓
Deployed; the running version is traceable to an exact commit
    ↓
─────────────── something breaks ───────────────
git bisect start; mark good and bad
    ↓
Git checks out midpoints; you test; ~10 steps over 1,000 commits
    ↓
The offending commit identified
    ↓
git revert <sha>  → a NEW commit undoing it, safe on shared history
    ↓
─────────────── someone panics ───────────────
"I ran reset --hard and lost two commits"
    ↓
git reflog → the hashes are still there → git switch -c recovery <sha>
    ↓
Nothing was actually lost, because commits are immutable
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Learn the graph — commits are immutable snapshots and branches are movable labels — then keep branches short-lived, use `--force-with-lease` instead of `--force`, revert rather than reset on shared history, and remember reflog exists.

---

## 19. Common Mistakes

- **Committing secrets**, then "removing" them in a later commit instead of rotating
- **`git push --force`** on a shared branch, discarding colleagues' commits
- **`reset --hard` on pushed commits** where a revert was needed
- **Branches open for weeks**, guaranteeing painful merges
- **Enormous unfocused commits** with a message like "fixes"
- **Committing generated files and dependencies**
- **Large binaries without LFS**, permanently bloating every clone
- **Never learning reflog or bisect**, and losing hours to problems either would solve in minutes
- **Submodules**, chosen before understanding what they cost
- **Full clones in CI** when a shallow one would do
- **Treating branch naming conventions as the strategy** while ignoring branch lifetime

---

## 20. Open Source Technologies

- **Git** itself — and reading `git help glossary` once is worth the time
- **lazygit**, **tig**, **gitui** — terminal interfaces that make the graph visible
- **delta** — dramatically better diff output
- **pre-commit** — run formatters, linters and secret scanners before a commit exists
- **gitleaks**, **trufflehog** — find secrets in the working tree and history
- **git-filter-repo** — the supported way to rewrite history when you must
- **Git LFS** — large binaries, if you cannot avoid them
- **commitizen**, **conventional-commits** — structured messages that generate changelogs

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Run `git log --oneline --graph --all` on a real repository and describe the shape out loud.
- [ ] Use `git add -p` for a whole day and notice how it changes your commits.
- [ ] Deliberately break something, run `reset --hard`, then recover it with `reflog`.
- [ ] Find a regression in a real repository using `git bisect`.
- [ ] Add a pre-commit secret scanner to one project.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
working dir → index → local repo → remote → CI → artefact → deploy
                        (a graph of immutable snapshots)
```

## 2. Request Flow

```text
Input       file changes staged deliberately
    ↓
Processing  hashed into blobs, trees and a commit; a branch label moves
    ↓
Output      an immutable, verifiable history that CI and deployment can act on
```

## 3. Real-World Usage

Git's reach extends well past source code: infrastructure, configuration, dashboards, documentation and machine learning pipelines all now live in repositories, because a reviewed, revertible history turned out to be valuable for anything that changes. GitOps takes this to its conclusion — the repository is the desired state of production, and the commit history is the deployment history.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A distributed database of immutable snapshots with movable branch labels |
| **Why does it exist?** | Because shared work needs history, and history needs to be cheap and local |
| **Where does it belong?** | Around everything textual — code, config, infrastructure, docs |
| **When should I use it?** | Always; the skill worth investing in is the graph, not the commands |

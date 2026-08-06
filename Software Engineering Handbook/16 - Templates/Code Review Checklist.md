# Code Review Checklist

> **In one line —** what to look for in a diff, in priority order; and the most valuable thing a reviewer does is refuse to approve a change too large to review.

| | |
|---|---|
| **Category** | Template |
| **Use for** | Every pull request, as a reviewer and as an author |
| **Related notes** | [Security Checklist](Security%20Checklist.md) · [GitHub](../13%20-%20DevOps%20and%20Delivery/GitHub.md) · [CI CD](../13%20-%20DevOps%20and%20Delivery/CI%20CD.md) · [Task Template](Task%20Template.md) · [Software Development Lifecycle](../01%20-%20Foundation/05%20-%20Software%20Development%20Lifecycle.md) |

---

## The one rule that matters most

```text
Reviewer effectiveness by diff size
    <  50 lines    real defects found, line by line
    < 200 lines    good review, some skimming
    < 500 lines    structural comments only; details missed
    > 500 lines    "LGTM" — approval by trust, not by reading
```

> [!IMPORTANT]
> **A 900-line pull request does not get reviewed; it gets approved.** Every checklist below is worthless if the diff is too large to hold in your head. The highest-value review comment in existence is *"please split this"* — and the second-highest is asking the author what they were trying to achieve, because a change whose purpose is unclear cannot be judged correct.

> [!TIP]
> **If you cannot split the change, split the review.** Read it twice: once for structure and intent, ignoring details; then once for correctness in the parts that matter. Say which pass you did in your comments, so the author knows what was actually examined.

---

## Priority order

Review in this order, and stop caring further down if something above is wrong.

```text
1. CORRECTNESS       does it do the right thing?
2. SECURITY          can it be abused?
3. DATA              is the migration safe? is data loss possible?
4. FAILURE           what happens when a dependency is slow or down?
5. TESTS             would a regression be caught?
6. CLARITY           will someone understand this in a year?
7. PERFORMANCE       only where it matters, and only if measured
8. STYLE             a linter's job, not yours
```

> [!CAUTION]
> **Comments about formatting in a review with an unhandled failure mode are worse than no review at all** — they signal that the code was read but not examined, and the author reasonably concludes it was checked. Automate style entirely (formatter, linter, in CI) so human attention goes where machines cannot help.

---

# ─────────── COPY FROM HERE ───────────

## 1. Before reading the code

- [ ] **Is the purpose clear?** Can I state what this change is for in one sentence?
- [ ] **Is it one change?** Or several unrelated things bundled together?
- [ ] **Is it small enough to review properly?** If not, say so now.
- [ ] **Is CI green?** A review of failing code is a waste of both people's time.
- [ ] **Does the description say WHY?** The diff already says what.

## 2. Correctness

- [ ] Does it actually do what the description claims?
- [ ] **Edge cases:** empty input, zero, one, null, maximum, negative, duplicate?
- [ ] **Off-by-one and boundary conditions** in loops, slices and ranges?
- [ ] **Are errors handled** — or swallowed silently in a `catch` that does nothing?
- [ ] **Is the happy path the only path that was thought about?**
- [ ] **Concurrency:** can two requests race here? Is anything shared and mutable?
- [ ] **Idempotency:** if this runs twice — a retry, a duplicate message — what happens?
- [ ] **Time zones, dates, DST, leap years** where dates are involved?
- [ ] **Money:** integers or decimals, never floats?

## 3. Security

- [ ] **Is user input validated** at the boundary, and validated server-side?
- [ ] **SQL:** parameterised queries, no string interpolation?
- [ ] **Output encoding:** could this render untrusted content as HTML or execute it?
- [ ] **Authorisation checked** — not just authentication? Can user A reach user B's record by changing an id?
- [ ] **Secrets:** none in code, config, logs or error messages?
- [ ] **Sensitive data in logs** — tokens, passwords, personal data, card numbers?
- [ ] **New dependency?** Is it maintained, and does it need to exist?
- [ ] **Does this add a new attack surface** — an endpoint, a file upload, a webhook?
- [ ] See [Security Checklist](Security%20Checklist.md) for anything security-shaped.

## 4. Data and migrations

- [ ] **Is the migration backward compatible?** Can the *previous* release run against it?
- [ ] **Anything destructive?** A dropped column or table cannot be rolled back.
- [ ] **Will it lock a large table?** How long, and at what traffic level?
- [ ] **Is there a backfill**, and is it batched rather than one enormous statement?
- [ ] **Indexes:** does the new query pattern have one? Created concurrently?
- [ ] **Can this be rolled back** without data loss?

## 5. Failure behaviour

- [ ] **Every external call has a timeout?**
- [ ] **What does the user see** when this dependency is slow, or down?
- [ ] **Retries:** present where safe, absent where not, with backoff?
- [ ] **Is a partial failure possible** — two writes where the second can fail?
- [ ] **Are errors observable?** Would we know this broke in production?
- [ ] **Is there a log line with enough context** to diagnose it — including the request id?

## 6. Tests

- [ ] **Would a regression here be caught?** That is the only question that matters.
- [ ] Do the tests test **behaviour**, or do they restate the implementation?
- [ ] **Is a failure case tested**, not only the happy path?
- [ ] **Would these tests fail if the code were wrong?** Try to imagine the bug they miss.
- [ ] **Any flaky or time-dependent tests** introduced (`sleep`, real clocks, real network)?
- [ ] **Are they fast enough** to keep the pipeline under ten minutes?

## 7. Clarity and design

- [ ] **Could I modify this in a year** without the author present?
- [ ] **Are names accurate?** A wrong name is worse than a vague one.
- [ ] **Do comments explain WHY**, not what? (What is the code's job.)
- [ ] **Is there duplication that will drift** — or premature abstraction that will need unpicking?
- [ ] **Does it fit the surrounding code's conventions**, even ones you dislike?
- [ ] **Is anything dead** — unused code, flags, config left behind?
- [ ] **Is the abstraction at one level**, or does high-level logic mix with byte handling?

## 8. Performance — only where it matters

- [ ] **Any query inside a loop?** (N+1 — the most common real performance defect.)
- [ ] **Is a result set bounded**, or does it grow with the table?
- [ ] **Is anything loaded into memory** that could be streamed?
- [ ] **Is a hot path doing work that could be cached** — and if cached, is the key scoped to the user or tenant?
- [ ] **Is there a measurement**, or is this speculation? (Speculative optimisation is a cost, not a benefit.)

## 9. Operational

- [ ] **Does anything need a config or environment change** to deploy? Is it documented?
- [ ] **Feature flag** for anything risky or user-visible?
- [ ] **Does this need a metric or an alert** it does not have?
- [ ] **Does it change a public API contract?** Versioned, or breaking?
- [ ] **Is there documentation to update** — a README, a runbook, an ADR?

# ─────────── COPY TO HERE ───────────

---

## How to write the comments

```text
UNHELPFUL                          HELPFUL
"this is wrong"                    "if `items` is empty this throws — line 42"
"use a map here"                   "a map would avoid the O(n²) — worth it if
                                    this list can be large; can it?"
"bad naming"                       "`data` — could this be `pendingInvoices`?"
"why did you do it this way?"      "I'd have reached for X — what made you
                                    choose Y? Genuinely asking."
```

```text
LABEL THE SEVERITY, so the author knows what blocks
    BLOCKING     must change before merge
    SUGGESTION   I would do it differently; your call
    NIT          trivial, feel free to ignore
    QUESTION     I do not understand this yet
    PRAISE       yes, actually — do this
```

> [!TIP]
> **Labelling severity is the single cheapest improvement to review culture.** Without it, every comment reads as a demand, authors argue about nits, and reviews become adversarial. With it, a reviewer can leave twelve thoughts of which one blocks, and everyone knows which. Add `PRAISE` deliberately — reviews that only ever criticise train people to dread them.

> [!CAUTION]
> **Review the code, not the person, and be careful with "why did you".** The same question phrased as "what led you to X?" gets an explanation; phrased as "why on earth did you X?" it gets defensiveness and a worse outcome. This is not politeness for its own sake — a defensive author stops volunteering the uncertainty you most need to hear about.

---

## For the author

```text
BEFORE REQUESTING REVIEW
    read your own diff, first          → you will find several things
    CI green
    small, and one purpose
    describe WHY, and how you verified it
    flag the parts you are unsure about — reviewers will look there
    leave your own comments on anything non-obvious

DURING REVIEW
    a question is not an attack
    "good catch" costs nothing
    if two comments disagree, get the reviewers to resolve it — not you
    if a comment reveals unclear code, fix the CODE, not the comment thread
```

> [!IMPORTANT]
> **Pointing at your own weak spots gets you a better review.** "I'm not confident about the locking in `reserve()`" directs attention where it is most valuable, and it is the opposite of a weakness — it is the behaviour of someone who wants the bug found now rather than in production.

---

## Common mistakes

- **Approving a diff too large to have read** — the most common failure, and the least discussed
- **Style comments** in a review that missed a missing timeout
- **No severity labels**, so nits read as blockers
- **Rubber-stamping** a colleague's work because they are senior, or nitpicking because they are junior
- **Reviewing only the diff**, without opening the surrounding code — many defects are only visible in context
- **Missing the migration** because it was a small file in a large change
- **Never asking "would a regression be caught?"** and instead counting test lines
- **Blocking on personal preference** rather than on a stated standard
- **Long review threads** where a five-minute conversation would resolve it
- **Reviewing while distracted**, which produces approval rather than review

---

## Personal Notes

*Your notes on using this checklist.*

- **Sections I always use:**
- **Bugs I have missed, and what would have caught them:**
- **Our team's additions:**

---

## Workbook Exercise

- [ ] Look at your last ten merged pull requests and record their sizes. Be honest about which were genuinely reviewed.
- [ ] Take one recent production bug and find which checklist item would have caught it. Add it if it is missing.
- [ ] Use severity labels on your next review and see whether the conversation changes.
- [ ] Add a pull request template that asks: what, why, and how it was verified.
- [ ] Check that formatting and linting are fully automated, so no human ever comments on them.

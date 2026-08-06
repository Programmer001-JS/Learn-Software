# Architecture Decision Record

> **In one line —** a short, dated, immutable note recording what you decided and why; its value is not the decision but the reasoning, which is the part that always gets forgotten.

| | |
|---|---|
| **Category** | Template |
| **Use for** | Any decision that is expensive to reverse |
| **Related notes** | [Universal Technology Learning Template](Universal%20Technology%20Learning%20Template.md) · [Architect Mindset](../00%20-%20Introduction/Architect%20Mindset.md) · [Task Template](Task%20Template.md) · [Git](../13%20-%20DevOps%20and%20Delivery/Git.md) |

---

## Why bother

```text
Eighteen months later, someone asks:
    "Why are we using Kafka here? A queue would have been simpler."
    ↓
Nobody who made the decision is still on the team
    ↓
Two options remain:
    guess, and keep it        → carry a constraint nobody understands
    guess, and replace it     → rediscover the reason the hard way
```

> [!IMPORTANT]
> **An ADR records the decision's *context*, and context is what expires invisibly.** The decision itself is usually visible in the code — the reasoning never is. Six months later the constraints that forced the choice may have vanished, and without a record nobody can tell whether the decision is still correct or merely still in place. That distinction is the entire point.

---

## What deserves one

```text
WRITE AN ADR FOR
    choosing a database, queue, or cloud provider
    a significant structural change — monolith to services, adding a cache tier
    a decision you argued about for more than an hour
    deliberately accepting a trade-off (we will oversell inventory occasionally)
    NOT doing something obvious, and why
    anything a future engineer would reasonably want to reverse

DO NOT WRITE ONE FOR
    naming conventions, formatting, library minor versions
    anything you could reverse in an afternoon
    decisions with no alternative worth considering
```

> [!TIP]
> **The best signal is disagreement: if two competent people argued, write it down.** The argument means there were real trade-offs, and the reasoning that resolved it is exactly what will be lost. A decision nobody questioned probably needs no record — though "we deliberately chose the boring option" is occasionally worth stating, because otherwise someone will assume nobody thought about it.

---

## Rules that make them work

```text
SHORT              one page. Two at most. Long ADRs do not get read or written.
IMMUTABLE          never edit a decision. Supersede it with a new one.
DATED and NUMBERED 0001, 0002, … in the repository, beside the code
STATUS              Proposed → Accepted → Superseded by ADR-00XX / Deprecated
HONEST              record the option you rejected AND what you gave up
IN VERSION CONTROL not a wiki that will be migrated and lost
```

> [!CAUTION]
> **Editing an ADR destroys its purpose.** The record is valuable precisely because it captures what was known *at that time* — including the things that later turned out to be wrong. Rewriting it produces a document that always looks correct and therefore teaches nothing. When circumstances change, write ADR-0014 that supersedes ADR-0009, and mark the old one. The pair together is the actual history.

---

## Where to keep them

```text
repo/
  docs/
    adr/
      0001-use-postgresql-as-primary-datastore.md
      0002-reject-microservices-for-now.md
      0003-accept-eventual-consistency-for-inventory-display.md
      0009-adopt-kafka-for-order-events.md
      0014-replace-kafka-with-sqs.md      ← supersedes 0009
```

> [!TIP]
> **Keep them in the repository, reviewed in a pull request like code.** That way the decision is discussed before it is made, the reviewers are the people affected, and the record cannot be separated from the system it describes. A wiki page survives about one company reorganisation.

---

# ─────────── COPY FROM HERE ───────────

# ADR-00XX — <Short decision title, stated as a decision>

| | |
|---|---|
| **Status** | Proposed / **Accepted** / Superseded by ADR-00XX / Deprecated |
| **Date** | YYYY-MM-DD |
| **Deciders** | <names — who is accountable for this> |
| **Consulted** | <who was asked> |
| **Supersedes** | <ADR-00XX, if applicable> |

---

## Context

*What situation forces a decision? What are the constraints?*

<Two to five paragraphs. State the facts, not the conclusion. Include the numbers that matter: current load, expected growth, team size, deadline, budget, regulatory constraint.>

<Be explicit about what is **unknown**. "We do not know whether write volume will grow 10× or 2×" is important context, because it explains why the decision favours reversibility.>

---

## Decision

*What are we doing?*

**We will <do the thing>.**

<One paragraph. Active voice, present tense, unambiguous. A reader should be able to act on this sentence alone.>

---

## Options considered

### Option A — <name>  ← chosen

- <what it is>
- **Pros:** <specific to our context, not generic>
- **Cons:** <what we are accepting>

### Option B — <name>

- <what it is>
- **Pros:**
- **Cons:**
- **Why rejected:** <the specific reason, in our situation>

### Option C — do nothing / keep the current approach

- **Why rejected:** <always include this option; sometimes it wins>

---

## Consequences

*What becomes true because of this decision?*

**Positive**
- <what gets easier>

**Negative**
- <what gets harder — be honest; this section is where credibility is earned>

**Neutral / follow-on work**
- <what this now obliges us to do: monitoring, migration, documentation, training>

---

## Reversibility

```text
COST TO REVERSE     <days / weeks / months / effectively permanent>
WHAT WOULD LOCK US IN   <data gravity, API contracts, team knowledge>
```

> <If reversal is cheap, say so — it lowers the stakes and should have lowered the deliberation. If it is expensive, that is the most important sentence in this document.>

---

## What would make us revisit this

*The trigger conditions. Write these down while you still remember them.*

- <a number: "if write volume exceeds X per second">
- <an event: "if we need multi-region writes">
- <a date: "review in 12 months if neither of the above happens">

---

## Notes

<Links to benchmarks, spikes, vendor comparisons, the Slack thread, the meeting. Anything that constitutes evidence.>

# ─────────── COPY TO HERE ───────────

---

## A worked example

```text
ADR-0003 — Accept eventual consistency for inventory display

STATUS   Accepted        DATE  2026-02-14

CONTEXT
    Product pages are 92% of traffic and must render in under 200 ms.
    Stock is held across 4 warehouses and sold on 3 channels.
    Showing exact live stock would require coordinating all warehouses on
    every page view — measured at 340 ms p99 in a spike, and it does not
    scale with traffic.
    Overselling currently occurs ~12 times per month and is handled by
    support with a refund and an apology, at roughly £4 per incident.

DECISION
    We will display availability from a cache with up to 30 seconds of
    staleness, and reserve stock atomically only at checkout.

OPTIONS
    A. Cached display + atomic reservation at checkout    ← chosen
    B. Live coordinated stock check on every page view
       rejected: 340 ms p99, and it gets worse with traffic
    C. Pessimistic reservation when an item enters a cart
       rejected: abandoned carts would hold stock; worse overselling
       in practice, plus a new expiry mechanism to build

CONSEQUENCES
    + product pages stay under 200 ms at any traffic level
    + no cross-warehouse coordination on the read path
    − overselling continues, and will rise roughly with order volume
    − support process must remain funded; this is now a DESIGN dependency
    → we must monitor the oversell rate as a product metric

REVERSIBILITY
    Weeks. The display path is isolated behind one service call.

REVISIT IF
    − oversell incidents exceed 100/month, or
    − a contractual commitment requires exact availability, or
    − review in 12 months
```

> [!TIP]
> **Notice what makes that example useful: numbers.** "340 ms p99 in a spike", "12 times per month", "£4 per incident". An ADR full of adjectives — "too slow", "quite risky" — cannot be re-evaluated later, because nobody can tell whether the situation changed. Measured facts age visibly; opinions do not.

---

## Common mistakes

- **Writing it after the fact**, as documentation rather than as a decision — the reasoning is already lost by then
- **Editing an accepted ADR** instead of superseding it
- **Omitting the rejected options**, which is where most of the value lives
- **Only listing advantages** — an ADR with no honest downside reads as advocacy and is not trusted
- **Too long**, so nobody writes the next one
- **Stored in a wiki** that will be migrated, reorganised and lost
- **No revisit trigger**, so the decision quietly becomes permanent
- **No reversibility assessment**, so a cheap decision gets three weeks of deliberation and an expensive one gets an afternoon
- **Writing them for everything**, which trains people to ignore them

---

## Personal Notes

*Your notes on using this template.*

- **Decisions I wish had an ADR:**
- **Adaptations I made:**
- **Where we keep ours:**

---

## Workbook Exercise

- [ ] Write an ADR for a decision your current system already made, from memory. Notice what you cannot reconstruct.
- [ ] Create `docs/adr/` in one repository and write ADR-0001.
- [ ] Find one architectural constraint nobody can explain. That is the ADR that was never written.
- [ ] Add a revisit trigger to a decision that is currently permanent by default.

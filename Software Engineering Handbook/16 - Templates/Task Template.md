# Task Template

> **In one line —** a structure for writing down a piece of work before starting it; and the section that matters most is the one stating how you will know it is done.

| | |
|---|---|
| **Category** | Template |
| **Use for** | Any task larger than an afternoon, and any task you will hand to someone else |
| **Related notes** | [Architecture Decision Record](Architecture%20Decision%20Record.md) · [Code Review Checklist](Code%20Review%20Checklist.md) · [Universal Technology Learning Template](Universal%20Technology%20Learning%20Template.md) · [CI CD](../13%20-%20DevOps%20and%20Delivery/CI%20CD.md) |

---

## Why write it down

```text
A task described in one line — "add search"
    ↓
Three days of work
    ↓
"That's not what I meant"
    ↓
Two more days
    ↓
Nobody can say whether it is finished, because nobody said what finished meant
```

> [!IMPORTANT]
> **The purpose is not documentation, it is forcing the thinking to happen before the work rather than during it.** Writing down what you will *not* do, how you will know it works, and what could go wrong takes fifteen minutes and routinely changes the approach. That is the whole return — a task description that turns out to be wrong on paper is far cheaper than one that turns out wrong in code.

---

## Scale the template to the task

```text
UNDER AN HOUR       just do it. Writing it down costs more than the work.
HALF A DAY          one line of goal, one line of "done when".
A FEW DAYS          the full template below.
MORE THAN A WEEK    the full template, PLUS split it into smaller tasks
                    → a task nobody can finish in a week is usually two tasks
                      and one unanswered question
```

> [!CAUTION]
> **A task estimated at "two or three weeks" is almost always hiding an unresolved decision.** Before starting, find the question you have not answered — which library, which data model, whose approval — and answer that first. Long estimates are usually uncertainty wearing a number.

---

# ─────────── COPY FROM HERE ───────────

# <Task title — a verb and an outcome>

| | |
|---|---|
| **Status** | Not started / In progress / Blocked / In review / Done |
| **Owner** | <one name — not a team> |
| **Size** | <hours / days / weeks> |
| **Priority** | <and why now, rather than later> |
| **Related** | <issue, ADR, design doc, incident> |

---

## Goal

*What outcome are we producing, and for whom?*

<One or two sentences. An outcome, not an activity. "Users can find a product by typing part of its name" — not "implement Elasticsearch".>

---

## Why now

*What is the cost of not doing this?*

<One or two sentences. If you cannot articulate this, the task may not be worth doing yet — and saying so is a valid conclusion.>

---

## Done when

*The acceptance criteria. Observable, not aspirational.*

- [ ] <something a person can verify by looking>
- [ ] <a number, if a number applies: "p95 under 300 ms">
- [ ] <the test that exists afterwards>
- [ ] <the documentation or runbook updated>

> **Write this section before writing any code.** If you cannot state how you would know it works, you do not yet know what you are building.

---

## Explicitly NOT in scope

*What someone might reasonably assume is included, and is not.*

- <the adjacent thing that is a separate task>
- <the nice-to-have deliberately deferred>
- <the edge case being accepted for now, and why>

> **This is the section that prevents the "that's not what I meant" conversation.** It is also the section people skip.

---

## Approach

*How, in enough detail that someone could disagree with it.*

```text
<the steps, or a small diagram>
```

<Note the parts you are unsure about — those are where review is most valuable.>

**Alternatives considered:** <one line each, and why not. If a decision here is expensive to reverse, it needs an [ADR](Architecture%20Decision%20Record.md) rather than a line in a task.>

---

## Dependencies and blockers

- **Needs from others:** <person or team, and what specifically>
- **Technical prerequisites:** <what must exist first>
- **Decisions still open:** <and who decides>

---

## Risks

| Risk | Likelihood | If it happens |
|---|---|---|
| <the thing most likely to go wrong> | | |
| <the thing that would hurt most> | | |

---

## How this ships

*How it reaches production safely — decided before the work, not after.*

- [ ] **Behind a feature flag?** <yes/no, and why>
- [ ] **Database migration?** <backward compatible? locking? backfill?>
- [ ] **Rollback plan:** <how, and how long it takes>
- [ ] **Monitoring:** <what metric or alert tells us it is working, or not>
- [ ] **Who needs to know** it shipped: <support, sales, docs, users>

---

## Notes and progress

*Append as you go. Do not rewrite history — this is where the useful detail lives.*

```text
YYYY-MM-DD  <what happened, what was learned, what changed>
YYYY-MM-DD  <the assumption that turned out to be wrong>
```

# ─────────── COPY TO HERE ───────────

---

## A worked example

```text
TITLE   Add product name search to the catalogue page

GOAL
    A customer can type part of a product name and see matching products,
    ranked sensibly, without waiting.

WHY NOW
    32% of catalogue sessions end without a product view. Support says
    "I couldn't find it" is the most common complaint this quarter.

DONE WHEN
    [ ] typing 3+ characters returns matches, including partial words
    [ ] p95 under 200 ms with the full production catalogue
    [ ] misspellings within one character still match
    [ ] tests cover: empty query, no results, special characters, 10k results
    [ ] a dashboard shows search volume and zero-result rate

NOT IN SCOPE
    − faceted filtering (separate task)
    − search across descriptions and reviews (later, once names work)
    − multi-language (single market for now)
    − personalised ranking (needs data we do not yet collect)

APPROACH
    PostgreSQL full-text search with a trigram index, not Elasticsearch.
    → the catalogue is 40k rows; a new search cluster is not justified yet
    Alternatives:
      Elasticsearch — better at scale, an entire new system to operate
      LIKE '%term%'  — no index, full scan, already tried and too slow

RISKS
    trigram index size on 40k rows          low       acceptable, measured at ~30 MB
    ranking quality is subjective            medium    ship behind a flag, measure
                                                       zero-result rate

HOW THIS SHIPS
    [x] feature flag: search_v1
    [x] migration: CREATE INDEX CONCURRENTLY — no lock
    [x] rollback: disable the flag; the index can stay
    [x] monitoring: search latency, zero-result rate, searches per session

NOTES
    2026-03-02  trigram index built in 8 s on a production-sized copy
    2026-03-03  "misspelling" criterion was vague — narrowed to a similarity
                threshold of 0.3, agreed with product
```

> [!TIP]
> **Notice what the example does: it kills a decision before it becomes work.** "PostgreSQL, not Elasticsearch, because the catalogue is 40k rows" is one line that saves a fortnight of cluster operations. Writing the approach down is what makes that reasoning visible enough to challenge — and if the reasoning is wrong, someone can say so in a comment rather than after the deployment.

---

## Common mistakes

- **No "done when"**, so the task ends by exhaustion rather than by completion
- **No "not in scope"**, guaranteeing a scope disagreement later
- **A goal that describes activity** ("implement Redis") rather than outcome
- **A team as the owner**, so nobody is accountable
- **No deployment plan**, so the flag, migration and rollback are improvised at the end
- **Acceptance criteria with no numbers** where numbers apply — "fast" is not a criterion
- **Notes rewritten** rather than appended, erasing the wrong assumption that was the useful part
- **A multi-week task not split**, hiding an unanswered question
- **An expensive architectural choice buried in "approach"** where it needed an ADR
- **Writing the template for a one-hour task**, which trains people to skip it entirely

---

## Personal Notes

*Your notes on using this template.*

- **Sections I always use:**
- **Sections I never use:**
- **Tasks that went wrong, and which section would have caught it:**

---

## Workbook Exercise

- [ ] Take a task you are currently working on and write its "done when" section. Notice whether it was clear.
- [ ] Write the "not in scope" section for the same task. Notice how much you were quietly assuming.
- [ ] Find a task in your backlog with no acceptance criteria and add them.
- [ ] Look at a task that overran and identify which section was missing.
- [ ] For your next task, decide the flag, migration and rollback plan *before* writing code.

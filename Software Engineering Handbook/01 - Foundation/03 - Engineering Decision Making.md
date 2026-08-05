# Engineering Decision Making

> **In one line —** a good decision is not the one that turns out well, it is the one made with a clear reason you can still defend when it turns out badly.

| | |
|---|---|
| **Category** | Foundation Concept |
| **Prerequisites** | [How Software Architects Think](02%20-%20How%20Software%20Architects%20Think.md) |
| **Related notes** | [Architecture Decision Record](../16%20-%20Templates/Architecture%20Decision%20Record.md) · [Problem First Technology Second](04%20-%20Problem%20First%20Technology%20Second.md) |

---

## 1. The problem

Technology decisions are usually made by whoever argues most confidently, by what someone read last week, or by what the loudest conference talk recommended. Months later nobody remembers *why* the choice was made, so nobody dares to revisit it — and the wrong decision becomes permanent.

Decision-making is a skill with a process, not a matter of taste.

---

## 2. The decision process

```text
1. State the problem      (not the solution)
    ↓
2. List the constraints   (budget, team, deadline, traffic, skills)
    ↓
3. Generate 2–4 options   (one of them is always "do nothing")
    ↓
4. Compare trade-offs     (what each one costs, not just what it gives)
    ↓
5. Decide
    ↓
6. Write down WHY         (this is the step everyone skips)
    ↓
7. Set a review trigger   ("revisit if traffic exceeds X")
```

> [!IMPORTANT]
> Step 6 is the entire value. A decision without a written reason cannot be re-evaluated later — it can only be inherited and resented.

---

## 3. Always include "do nothing"

The simplest option is a real option and it is often correct. Adding Kafka, Kubernetes or a second service has a permanent operational cost that the exciting version of the proposal never mentions.

| Option | Gain | Cost |
|---|---|---|
| Do nothing | No new complexity | The problem remains |
| Simple fix | Solves it today | May not scale |
| Proper solution | Solves it for years | Weeks of work, new component to operate |

---

## 4. Reversible vs irreversible

Amazon calls these **two-way doors** and **one-way doors**.

```text
Two-way door  →  you can walk back through  →  decide in an hour
One-way door  →  you cannot                 →  decide in a week, write an ADR
```

- **Two-way:** a library, a code convention, a CI tool, a hosting provider for a small service
- **One-way:** the data model, service boundaries, the auth model, the public API contract, a database engine holding years of data

> [!CAUTION]
> Treating a one-way door like a two-way door is how teams end up rewriting a system. Treating a two-way door like a one-way door is how teams spend three weeks choosing a logging library.

---

## 5. Compare on the same criteria

Pick the criteria **before** looking at the options, otherwise you will unconsciously pick criteria that favour the option you already like.

| Criterion | Option A | Option B |
|---|---|---|
| Fits the constraint? | | |
| Team already knows it? | | |
| Operational cost | | |
| Failure behaviour | | |
| Cost to reverse | | |
| Licence / vendor risk | | |

---

## 6. Common biases to catch in yourself

- **Résumé-driven development** — choosing what looks good on a CV rather than what fits
- **Novelty bias** — new is assumed better; usually it is just less understood
- **Sunk cost** — "we already spent two months on it" is not a reason to spend two more
- **Authority bias** — "Netflix does it" ignores that Netflix has different constraints and 2,000 engineers
- **Availability bias** — choosing the tool you read about most recently

> [!TIP]
> A quick test for bias: can you argue the opposing option convincingly? If not, you have not understood it well enough to reject it.

---

## 7. The cost that is always forgotten

Every new component you add costs you, forever:

```text
New component
    ↓
must be deployed, monitored, backed up, secured, upgraded,
documented and understood by everyone who joins later
```

This is why "we'll just add Redis" is never *just* adding Redis. The right question is not "is it useful?" but **"is it useful enough to pay for it every month for years?"**

---

## 8. Decide at the last responsible moment

Deciding too early means deciding with the least information you will ever have. Deciding too late blocks the team. The right moment is the last one at which the decision does not yet cost you anything to delay.

---

## 9. Mental Model

> [!NOTE]
> **A decision is a bet, and an ADR is the betting slip.**

You cannot control whether the bet wins. You can control whether, when the result comes in, you can see exactly what you knew, what you assumed and why you chose — which is the only way to bet better next time.

---

## 10. Key Takeaway

> [!IMPORTANT]
> Judge decisions by the quality of the reasoning at the time, not by the outcome — and always write the reasoning down.

---

## 11. Common Mistakes

- **Choosing the solution before defining the problem**
- **Comparing only one option against nothing**
- **Ignoring the permanent operational cost of a new component**
- **Never writing down why**, so the decision cannot be revisited
- **Never setting a review trigger**, so an outgrown decision stays forever

---

## 12. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 13. Workbook Exercise

- [ ] Take one technology decision you have already made and write it up as an ADR after the fact.
- [ ] For that decision, list the option you did *not* choose and argue for it convincingly.
- [ ] Classify three current decisions in your project as one-way or two-way doors.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Problem
    ↓
Constraints
    ↓
Options  (including "do nothing")
    ↓
Trade-off comparison
    ↓
Decision  →  ADR  →  review trigger
```

## 2. Request Flow

```text
Input       a problem plus its constraints
    ↓
Processing  generate options, compare on fixed criteria, check for bias
    ↓
Output      a decision, a written reason, and a condition to revisit it
```

## 3. Real-World Usage

**Amazon** built the one-way/two-way door distinction into how it operates: most decisions are delegated and made quickly, while a small set of irreversible ones get deliberate, documented review. Speed on the reversible ones is what makes the care on the irreversible ones affordable.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A repeatable process for making and recording technical choices |
| **Why does it exist?** | Because undocumented decisions cannot be re-evaluated |
| **Where does it belong?** | Before implementation, at every architectural fork |
| **When should I use it?** | Whenever the decision is hard to reverse or affects more than one team |

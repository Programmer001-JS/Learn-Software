# How Software Architects Think

> **In one line —** they think in constraints, consequences and failure, not in features.

| | |
|---|---|
| **Category** | Foundation Concept |
| **Prerequisites** | [What Is Software Architecture](01%20-%20What%20Is%20Software%20Architecture.md) |
| **Related notes** | [Architect Mindset](../00%20-%20Introduction/Architect%20Mindset.md) · [Engineering Decision Making](03%20-%20Engineering%20Decision%20Making.md) |

---

## 1. The core habit — start from constraints

A developer starts from the feature. An architect starts from what is **not allowed to change**.

```text
Constraints        (budget, team size, deadline, traffic, compliance, latency)
    ↓
Options that survive those constraints
    ↓
Trade-off comparison
    ↓
Decision + written reason
```

Most technology arguments are actually constraint disagreements in disguise. Once the constraints are on the table, the number of realistic options usually drops to two.

---

## 2. They ask "and then what?"

Every proposal gets pushed forward in time until it breaks.

| Proposal | And then what? |
|---|---|
| "We'll store uploads on the server disk" | And then you add a second server — which one has the file? |
| "We'll cache it" | And then the data changes — who clears the cache? |
| "We'll call the payment API directly" | And then it is down for 10 minutes — what do users see? |
| "We'll use one database for everything" | And then the reporting query locks the users table at 9am |

> [!TIP]
> Asking "and then what?" three times in a row finds almost every design flaw before a line of code is written.

---

## 3. They think in failure, not success

The happy path is the easy part. Architects spend their attention on the unhappy ones:

- What if this component is **down**?
- What if it is **slow** instead of down? *(Slow is worse — it takes the callers down with it.)*
- What if the same message arrives **twice**?
- What if the process dies **halfway through**?
- What if the input is **hostile**?

> [!CAUTION]
> A system that has no answer for these questions is not "simple" — it is a system whose failure behaviour is simply unknown.

---

## 4. They separate the reversible from the irreversible

```text
Is this decision easy to undo?
    ↓                      ↓
   YES                    NO
    ↓                      ↓
Decide fast,         Slow down, compare
move on              alternatives, write an ADR
```

Examples of irreversible-ish decisions: the data model, service boundaries, the auth mechanism, the public API contract, the choice between SQL and document storage.

---

## 5. They optimise for the whole system, not the part

A change that makes one service 20% faster but makes three others harder to change is a **loss**. Architects are permanently zoomed out; that is the entire difference in perspective.

```text
Developer view          Architect view

  [ my service ]        [ svc A ] → [ svc B ] → [ svc C ]
       ↓                     ↘        ↓        ↙
   make it good                 [ database ]
                             where does it break?
```

---

## 6. They distrust their own predictions

Nobody knows what the product will need in two years. So instead of predicting, architects buy **optionality**: keep boundaries clean so a component can be replaced, avoid coupling to a vendor where the cost is low, and delay decisions until the last responsible moment.

> [!IMPORTANT]
> The goal is not to be right about the future. It is to be **cheap to be wrong**.

---

## 7. They quantify

"Fast", "a lot of users" and "scalable" are not engineering terms. Architects convert them into numbers before designing anything:

- How many requests per second — at peak, not average?
- How much data today, and how much in a year?
- Acceptable latency: 100 ms or 2 seconds?
- Acceptable downtime: 5 minutes a month or 5 hours?
- Is stale data acceptable, and for how long?

The answers usually reveal that the "hard scaling problem" is one server and a cache.

---

## 8. They know the numbers of the machine

Rough orders of magnitude an architect carries in their head:

| Operation | Roughly |
|---|---|
| CPU cache read | ~1 ns |
| RAM read | ~100 ns |
| SSD read | ~100 µs |
| Network call, same data centre | ~0.5 ms |
| Database query | ~1–10 ms |
| Network call across the internet | ~50–150 ms |

> [!IMPORTANT]
> A network call is roughly **a million times** slower than a memory read. Almost every architectural performance decision follows from this one fact.

---

## 9. They design boundaries, not code

The question is never "where do I put this function?" but **"who is allowed to know this?"** Good boundaries mean a change stays inside one box. Bad boundaries mean every change ripples outward.

---

## 10. Mental Model

> [!NOTE]
> **A developer is a driver. An architect is the city traffic planner.**

The driver optimises their own trip. The planner accepts that any individual trip may be slightly slower, as long as the whole city does not deadlock at 5pm.

---

## 11. Key Takeaway

> [!IMPORTANT]
> Architects think in constraints, consequences and failure modes — and they write the reasoning down, because the reasoning is the part that cannot be rediscovered from the code.

---

## 12. Common Mistakes

- **Designing from technology enthusiasm** instead of from constraints
- **Only designing the happy path**
- **Treating every decision as equally important**, so the big ones get the same ten minutes as the small ones
- **Refusing to decide** — endless analysis is itself a decision, and usually a bad one
- **Designing alone** — architecture that the team does not understand will not be implemented

---

## 13. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 14. Workbook Exercise

- [ ] Take one feature you are building and ask "and then what?" three times in writing.
- [ ] List the five failure questions from section 3 for one component of your system, and answer each in one sentence.
- [ ] Convert one vague requirement of yours ("it should be fast") into three numbers.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Business goal
    ↓
Constraints  →  Quality attributes  →  Failure scenarios
    ↓
Options  →  Trade-offs  →  Decision (written down)
    ↓
Implementation
```

## 2. Request Flow

```text
Input       a vague requirement
    ↓
Processing  quantify it, list constraints, push it forward in time
    ↓
Output      a decision with a documented reason
```

## 3. Real-World Usage

**Netflix** designed on the assumption that servers fail constantly, and built Chaos Monkey to kill their own production instances on purpose. That is failure-first thinking turned into an engineering practice.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A way of reasoning about systems, based on constraints and consequences |
| **Why does it exist?** | Because local optimisation produces globally fragile systems |
| **Where does it belong?** | Before any technology choice |
| **When should I use it?** | Every time a decision would be expensive to reverse |

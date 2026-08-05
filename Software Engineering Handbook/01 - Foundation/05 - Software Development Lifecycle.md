# Software Development Lifecycle

> **In one line —** the path a piece of software takes from an idea in someone's head to running code that people depend on, and back again.

| | |
|---|---|
| **Category** | Foundation Concept |
| **Abbreviation** | SDLC |
| **Related notes** | [Documentation Driven Development](07%20-%20Documentation%20Driven%20Development.md) · [CI CD](../13%20-%20DevOps%20and%20Delivery/CI%20CD.md) · [Deployment Strategies](../13%20-%20DevOps%20and%20Delivery/Deployment%20Strategies.md) |

---

## 1. Short Definition

The Software Development Lifecycle is the **sequence of stages** software passes through: understanding the need, designing, building, testing, releasing, operating and improving it. Every team follows some version of it, whether they name it or not.

---

## 2. Purpose

To make sure the thing that gets built is the thing that was needed, that it works, that it reaches users safely, and that it keeps working. Skipping stages does not remove the work — it moves it to a more expensive moment.

---

## 3. The stages

```text
1. Requirements    what problem, for whom, and how do we know it worked?
    ↓
2. Design          architecture, data model, interfaces
    ↓
3. Implementation  writing the code
    ↓
4. Testing         does it do what we said, and what happens when it does not?
    ↓
5. Deployment      getting it to production safely
    ↓
6. Operation       monitoring, alerting, incidents, backups
    ↓
7. Maintenance     bug fixes, dependency updates, improvements
    ↓
   back to 1
```

> [!IMPORTANT]
> The arrow from 7 back to 1 is the important one. Software is never "finished" — it is either being maintained or being abandoned.

---

## 4. The cost curve

The cost of fixing a mistake grows by roughly an order of magnitude at every stage.

| Mistake found in | Relative cost to fix |
|---|---|
| Requirements | 1× |
| Design | ~5× |
| Implementation | ~10× |
| Testing | ~20× |
| Production | ~100× |

> [!CAUTION]
> This is the entire economic argument for design and code review. Time spent thinking before building is not overhead — it is the cheapest place to be wrong.

---

## 5. Waterfall vs Agile

```text
WATERFALL              all requirements → all design → all code → all testing → release
                       ↓
                       one huge bet, feedback arrives at the very end

AGILE / ITERATIVE      small slice through every stage, repeatedly
                       ↓
                       feedback every 1–2 weeks, direction can change
```

| | Waterfall | Agile |
|---|---|---|
| Feedback | Once, at the end | Continuous |
| Handles changing requirements | Badly | Well |
| Works when | Requirements truly are fixed (regulated, hardware-bound) | Almost everywhere else |
| Main risk | Building the wrong thing perfectly | Losing sight of the overall architecture |

> [!TIP]
> Agile does not mean "no design". It means designing the architecture early and the details continuously.

---

## 6. Architecture Position

The SDLC is the **process** wrapped around the system, not a part of the system.

```text
┌──────────────────── SDLC ────────────────────┐
│                                               │
│  Requirements → Design → Build → Test         │
│                                    ↓          │
│                              Deploy           │
│                                    ↓          │
│   ┌─────────── running system ──────────┐     │
│   │  Frontend → API → Database          │     │
│   └─────────────────────────────────────┘     │
│                                    ↓          │
│                       Monitor → Learn ────────┼──► back to Requirements
└───────────────────────────────────────────────┘
```

---

## 7. Where each handbook section fits

| Stage | Relevant folders |
|---|---|
| Requirements | 01 - Foundation |
| Design | 01, 07 - Design Patterns, 15 - System Design |
| Implementation | 05, 06, 07, 08, 11 |
| Testing | 07, 14 - Load Testing |
| Deployment | 13 - DevOps and Delivery |
| Operation | 12 - Cloud, 14 - Monitoring and Alerting |
| Security | 09 — *every* stage, not one of them |

---

## 8. Real World Example

- **Google** requires code review on essentially every change, moving defect discovery from stage 5 back to stage 3.
- **Amazon** deploys thousands of times per day; that is only possible because testing and deployment are automated stages rather than manual events.
- **Aerospace and medical software** still use waterfall-like processes, because a failed deployment cannot be rolled back with `git revert`.

---

## 9. Mental Model

> [!NOTE]
> **SDLC = building a house.**

Requirements are talking to the family about how they live. Design is the blueprint. Implementation is construction. Testing is inspection. Deployment is moving in. Operation is heating, plumbing and repairs — and it lasts far longer than construction did.

---

## 10. Key Takeaway

> [!IMPORTANT]
> Every stage you skip does not disappear; it reappears later at ten times the cost.

---

## 11. Common Mistakes

- **Starting to code before the problem is clear** — the most expensive mistake in software
- **Treating deployment as an event** instead of an automated, repeatable process
- **Treating testing as a phase** rather than something written alongside the code
- **Forgetting operation** — building something nobody can monitor, back up or debug
- **Believing "agile" means no planning**

---

## 12. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 13. Workbook Exercise

- [ ] Map your own current workflow onto the seven stages. Which stage is weakest?
- [ ] Take your last bug in production and identify at which stage it could have been caught.
- [ ] Write, in one sentence each, how you would deploy and how you would roll back your current project.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Requirements → Design → Implementation → Testing → Deployment → Operation → Maintenance
      ↑                                                                          │
      └──────────────────────── feedback ────────────────────────────────────────┘
```

## 2. Request Flow

```text
Input       a user need
    ↓
Processing  design, build, test, release
    ↓
Output      working software in production, plus what you learned from it
```

## 3. Real-World Usage

**Amazon** automated testing and deployment so thoroughly that deploying became routine rather than risky. Reducing the cost of one lap around the lifecycle is what allows them to do it thousands of times a day.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The stages software moves through, from need to operation |
| **Why does it exist?** | Because mistakes get exponentially more expensive the later they are found |
| **Where does it belong?** | Around the whole system, as process rather than component |
| **When should I use it?** | Always — the only choice is whether it is explicit or accidental |

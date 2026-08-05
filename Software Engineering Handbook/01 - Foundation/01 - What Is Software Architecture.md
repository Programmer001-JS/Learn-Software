# What Is Software Architecture

> **In one line —** architecture is the set of decisions that are expensive to change later.

| | |
|---|---|
| **Category** | Foundation Concept |
| **Prerequisites** | None |
| **Related notes** | [Architect Mindset](../00%20-%20Introduction/Architect%20Mindset.md) · [How Software Architects Think](02%20-%20How%20Software%20Architects%20Think.md) · [Architecture Principles](08%20-%20Architecture%20Principles.md) |

---

## 1. Short Definition

Software architecture is the **high-level structure of a system**: which parts exist, what each one is responsible for, how they communicate, and which rules everything must follow. It is the shape of the system, not the code inside it.

---

## 2. What it is not

> [!WARNING]
> Architecture is **not** a diagram, a folder structure, or a framework choice. Those are *outputs* of architecture. Architecture is the set of **decisions** behind them.

- Not "we use React and FastAPI" — that is a tech stack
- Not `/controllers`, `/services`, `/models` — that is a folder convention
- Not a picture in Miro — that is documentation of architecture

---

## 3. The defining test

A decision belongs to architecture if changing it later would be **painful, slow or risky**.

| Decision | Cost to change | Architecture? |
|---|---|---|
| Variable naming style | Minutes | No |
| Which HTTP library | An afternoon | No |
| Which UI framework | Weeks | Borderline |
| Relational vs document database | Months | **Yes** |
| Monolith vs microservices | Months to years | **Yes** |
| How users authenticate | Months, plus a security risk | **Yes** |
| Public API contract | Breaks every client | **Yes** |

---

## 4. What architecture actually decides

- **Components** — what parts exist (frontend, API, worker, cache, database)
- **Responsibilities** — what each part is allowed to know and do
- **Communication** — how they talk (HTTP, queue, direct call, event)
- **Data ownership** — who owns which data and who may write it
- **Boundaries** — what is inside a component and what is forbidden to cross
- **Cross-cutting rules** — auth, logging, error handling, validation

---

## 5. Architecture Position

Architecture is not a layer in the system — it is the description of **all** the layers and the lines between them.

```text
┌─────────────────────────────────────────┐
│  ARCHITECTURE = the boxes and the arrows │
├─────────────────────────────────────────┤
│                                          │
│   Browser                                │
│       ↓  HTTPS                           │
│   Frontend                               │
│       ↓  REST                            │
│   API  ────────► Queue ────► Worker      │
│       ↓                        ↓         │
│   Cache                    Storage       │
│       ↓                                  │
│   Database                               │
│                                          │
└─────────────────────────────────────────┘
```

> [!IMPORTANT]
> The **arrows** are the architecture. Anyone can list components; the value is in knowing exactly who may call whom, and who may not.

---

## 6. Why it exists

Without explicit architecture, every developer makes their own local decision. Each one is reasonable alone, and together they produce a system where everything depends on everything else, no change is safe, and nobody can explain how it works.

```text
No architecture
    ↓
Every part talks to every other part
    ↓
One change breaks three unrelated things
    ↓
Nobody dares to touch it
    ↓
Rewrite
```

---

## 7. Real World Example

- **Netflix** — split into hundreds of services so a failure in "recommendations" never stops "play video". That is an architectural decision, not a coding one.
- **Instagram** — ran an enormous user base on a deliberately simple Django + PostgreSQL architecture for years, because their bottleneck was images, not application logic.
- **Amazon** — the famous internal rule that every team must expose its data only through a service interface. One architectural rule created AWS.

---

## 8. Quality attributes

Architecture is mostly chosen to satisfy **non-functional** requirements. Features can be built in almost any architecture; these cannot be retrofitted cheaply:

- **Performance** — how fast under normal load
- **Scalability** — what happens at 100× the traffic
- **Availability** — what happens when a part dies
- **Security** — what an attacker can reach
- **Maintainability** — how long a new developer needs to be productive
- **Cost** — what it costs to run every month

> [!TIP]
> When someone asks "what is the best architecture", the honest answer is a question: **"best at which of these six?"** You cannot maximise all of them at once.

---

## 9. Levels of architecture

```text
System architecture      how services, databases and queues fit together
    ↓
Application architecture how one service is structured internally (layers, DDD)
    ↓
Code architecture        how modules, classes and functions relate
```

All three matter, but they fail differently: bad code architecture is annoying, bad system architecture is expensive.

---

## 10. Mental Model

> [!NOTE]
> **Architecture = the structural plan of a building.**

The plan decides where the load-bearing walls are, where the plumbing runs, how many floors are possible. You can repaint any room whenever you like — that is code. Moving a load-bearing wall after the building exists is a different kind of problem entirely — that is architecture.

---

## 11. Key Takeaway

> [!IMPORTANT]
> Architecture is the set of decisions that are expensive to change — so make them consciously, write down why, and keep them as few as possible.

---

## 12. Common Mistakes

- **Confusing architecture with technology** — choosing tools before understanding the problem
- **Designing for imaginary scale** — distributed complexity for a hundred users
- **Copying big-company architecture** — Netflix's design solves Netflix's problems, not yours
- **Treating it as a one-time phase** — architecture evolves, or it dies
- **Never writing it down** — undocumented architecture exists only in one person's head

---

## 13. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 14. Workbook Exercise

- [ ] Draw the architecture of a project you have built or used — boxes and arrows only.
- [ ] List three decisions in it that would be expensive to change today.
- [ ] For one of them, write down what you would do differently and what it would cost.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Requirements  (what the system must do)
    ↓
Constraints   (budget, team, deadline, traffic)
    ↓
ARCHITECTURE  (components, boundaries, communication)
    ↓
Technology    (React, FastAPI, PostgreSQL)
    ↓
Code
```

## 2. Request Flow

```text
Input       a business problem and its constraints
    ↓
Processing  decide components, boundaries and communication
    ↓
Output      a structure that survives change
```

## 3. Real-World Usage

**Amazon** required every team to communicate only through service interfaces, with no shared databases. That single architectural constraint made teams independently deployable — and eventually became AWS.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The high-level structure and the rules that hold a system together |
| **Why does it exist?** | Because uncoordinated local decisions produce unmaintainable systems |
| **Where does it belong?** | Above technology, below business requirements |
| **When should I use it?** | Whenever a decision would be expensive to reverse |

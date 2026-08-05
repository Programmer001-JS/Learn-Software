# Software Engineering Mindset

> **In one line —** engineering is not writing code that works, it is being able to explain why it will keep working.

| | |
|---|---|
| **Category** | Foundation Concept |
| **Prerequisites** | The rest of folder 01 |
| **Related notes** | [Architect Mindset](../00%20-%20Introduction/Architect%20Mindset.md) · [Learning Philosophy](../00%20-%20Introduction/Learning%20Philosophy.md) |

---

## 1. Programming vs engineering

| | Programming | Engineering |
|---|---|---|
| **Goal** | Make it work | Make it keep working |
| **Scope** | This machine, today | Many machines, for years |
| **Measure** | It runs | It runs under load, under failure, under new people |
| **Deliverable** | Code | Code, tests, docs, deployment, monitoring |

Programming is a part of engineering, the way laying bricks is a part of building a house.

---

## 2. Measure before you optimise

Human intuition about performance is reliably wrong. The bottleneck is almost never where it feels like it is.

```text
Feels slow
    ↓
MEASURE  ← profiler, timing, logs, database EXPLAIN
    ↓
Find the actual bottleneck   (usually one query or one N+1 loop)
    ↓
Fix that one thing
    ↓
Measure again
```

> [!CAUTION]
> Optimising without measuring is guessing with extra steps. It usually makes the code harder to read and the program no faster.

---

## 3. Understand the layer below

Almost every hard bug is explained one level below where it appears: a slow endpoint is a missing index, a memory leak is an unbounded cache, a random failure is a race between threads.

```text
Your code
    ↓
Framework / library
    ↓
Runtime / interpreter
    ↓
Operating system
    ↓
Hardware        ← "why is this slow" usually gets answered here
```

You do not need mastery of every layer — you need enough to know **which layer to look at**.

---

## 4. Debugging is a method, not a talent

```text
1. Reproduce it reliably       ← if you cannot, you cannot fix it, only guess
    ↓
2. Read the actual error       ← the whole message, the whole stack trace
    ↓
3. Form ONE hypothesis
    ↓
4. Test it — change one thing
    ↓
5. It was wrong? Back to 3, with new information
```

> [!TIP]
> The most common debugging failure is changing several things at once. When it starts working, you have learned nothing and it will come back.

---

## 5. Read more than you write

You will spend far more time reading code — your own from six months ago, your colleagues', a library's source — than writing it. Reading skill is therefore the higher-leverage one, and it is trainable: pick a small open-source project and read it until you can explain its structure.

---

## 6. Take ownership of the whole path

> [!IMPORTANT]
> "It works on my machine" is the boundary between a programmer and an engineer. An engineer owns the change until it is running correctly in production, and cares whether anyone can tell when it stops.

That means caring about: how it is deployed, how it fails, how it is monitored, how it is rolled back, and how the next person will understand it.

---

## 7. Simplicity is a discipline

Anyone can build something complicated. Building the *simple* version requires understanding the problem well enough to know what can be left out — which is much harder and much more valuable.

```text
Complex solution  →  usually a sign the problem is not yet understood
Simple solution   →  usually arrives after understanding, not before
```

---

## 8. Be honest about what you know

Saying "I don't know, I'll check" costs three seconds. Guessing confidently costs a day of someone else's work, and eventually your credibility. Precision about the edge of your own knowledge is a technical skill, not a personality trait.

---

## 9. Write for the next person

The next person to read your code has none of your context, and there is a good chance the next person is you. Clear naming, small functions and a comment explaining *why* the strange line exists are not politeness — they are the mechanism that lets a system survive its authors.

---

## 10. Continuous learning, structured

The field changes constantly at the surface and very slowly underneath. HTTP, TCP, relational databases, caching, queues and operating systems have been the same for decades, while frameworks come and go.

> [!TIP]
> Invest most of your learning time in the slow layer. It compounds. Framework knowledge depreciates.

---

## 11. Mental Model

> [!NOTE]
> **A programmer builds a bridge that holds. An engineer can tell you how much weight it holds, what happens in a storm, how long it lasts, and how you would know before it fell.**

---

## 12. Key Takeaway

> [!IMPORTANT]
> Engineering is accountability for the whole life of the system, not authorship of the code that started it.

---

## 13. Common Mistakes

- **Optimising without measuring**
- **Fixing symptoms instead of causes** — the bug returns wearing a different hat
- **Changing several things at once while debugging**
- **Stopping at "it works"** without asking how it fails
- **Chasing frameworks** while ignoring fundamentals
- **Pretending to know** instead of checking

---

## 14. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 15. Workbook Exercise

- [ ] Take your last performance assumption and actually measure it. Were you right?
- [ ] Take your last bug and write down which layer the real cause lived in.
- [ ] Open a file you wrote six months ago and note every place where you had to reconstruct your own reasoning.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Problem understanding
    ↓
Design  →  Code  →  Tests  →  Deployment  →  Monitoring
    ↓                                            │
    └────────────── ownership spans all of it ───┘
```

## 2. Request Flow

```text
Input       a problem, and constraints you may not like
    ↓
Processing  understand → design → build → measure → verify
    ↓
Output      a system that keeps working, and a person who can explain why
```

## 3. Real-World Usage

**Google's SRE practice** makes engineers responsible for the production behaviour of what they build, with explicit reliability targets. Tying authorship to operational consequences is what turns coding into engineering at scale.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The professional discipline around writing software |
| **Why does it exist?** | Because working code and reliable systems are not the same thing |
| **Where does it belong?** | Across the entire lifecycle, not one stage of it |
| **When should I use it?** | Whenever anyone other than you will depend on the result |

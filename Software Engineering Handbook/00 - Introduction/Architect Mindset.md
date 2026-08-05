# Architect Mindset

> **In one line —** a developer asks "how do I build this?", an architect asks "what happens to this in two years, under load, when it breaks, and when someone else maintains it?"

---

## The shift

The distance between a developer and an architect is not the amount of code they can write. It is the number of **consequences** they can see before writing it.

| | Developer thinking | Architect thinking |
|---|---|---|
| **Question** | How do I make this work? | What does this cost me later? |
| **Time horizon** | This sprint | Two years |
| **Success** | The feature works | The system still works when everything changes |
| **Focus** | Code | Boundaries between code |
| **Failure** | A bug | A design that cannot be undone |

> [!IMPORTANT]
> Both roles are necessary. An architect who cannot build produces diagrams nobody can implement. The mindset is an addition to engineering skill, never a replacement for it.

---

## 1. Every decision is a trade-off

There is no free choice in software. Adding a cache buys speed and pays with staleness. Splitting a monolith buys independence and pays with network calls, debugging pain and operational overhead.

```text
Decision
    ↓
What do I gain?     →  the reason you are tempted
    ↓
What do I pay?      →  the part beginners skip
    ↓
Can I undo it?      →  the part that actually matters
```

---

## 2. Reversibility is the real question

Some decisions are cheap to change and some are nearly permanent. Architects spend their attention proportionally.

- **Cheap to reverse** — a library, a code style, a framework version, a deployment tool. Decide fast, move on.
- **Expensive to reverse** — the database, the data model, the service boundaries, the auth model, the public API contract.

> [!TIP]
> Spend an hour on a reversible decision and a week on an irreversible one. Most teams do the exact opposite.

---

## 3. Boring technology is a feature

New technology carries hidden costs: no answers when it breaks, few people who know it, unstable APIs, unknown failure modes. **PostgreSQL is boring, and that is precisely why it is a good choice.**

Choose exciting technology only where it gives you an advantage you actually need, and pay the novelty cost consciously.

---

## 4. Design for the load you have

The most common failure of ambitious engineers is building for a million users while having a hundred. Premature distributed architecture produces systems that are hard to build, hard to debug and slower than the simple version they replaced.

```text
100 users        →  one server, one database
10,000 users     →  add a cache, add a read replica
1,000,000 users  →  now the hard architecture starts to earn its cost
```

> [!CAUTION]
> Scale you do not have is not a requirement — it is a fantasy that costs real time. Design so that scaling is *possible* later, not so that it is *solved* now.

---

## 5. Think in failures

A developer asks what happens when the code runs. An architect asks what happens when it does not:

- What if the database is down?
- What if this request takes 30 seconds instead of 30 milliseconds?
- What if the same request arrives twice?
- What if the server dies exactly halfway through?
- What if the third-party API returns garbage?

Every distributed system is a set of answers to those five questions. Systems that ignore them work perfectly in development and fail in production.

---

## 6. Optimise for the reader, not the writer

Code is written once and read hundreds of times, including by you in six months, when you have forgotten everything. Clever code that saves you ten minutes today costs the team hours later.

> [!IMPORTANT]
> The best architecture is the one the next person can understand without asking you.

---

## 7. Understand the layer below the one you work on

You do not need to write assembly, but you need to know that a database read touches a disk, that a network call is a thousand times slower than a memory read, and that RAM is finite. Almost every serious performance problem is explained one layer below where it appears.

```text
Your code
    ↓
Runtime / Interpreter
    ↓
Operating System
    ↓
Hardware  ← the answer to "why is it slow" usually lives here
```

That is why this handbook starts with [02 - Computer Science Fundamentals](../02%20-%20Computer%20Science%20Fundamentals) before touching a single framework.

---

## 8. Document the "why", not the "what"

Code already shows *what* it does. It can never show *why* it was done that way, which alternatives were rejected, or which constraint forced the ugly part. That knowledge lives only in people's heads and leaves when they do.

Use the [Architecture Decision Record](../16%20-%20Templates/Architecture%20Decision%20Record.md) template to capture it.

---

## The mindset in one paragraph

> [!IMPORTANT]
> An architect assumes that requirements will change, traffic will grow unevenly, components will fail, and someone else will maintain the result. Their job is not to predict the future — it is to build something that survives being wrong about it.

---

## Related

- [What Is Software Architecture](../01%20-%20Foundation/01%20-%20What%20Is%20Software%20Architecture.md)
- [How Software Architects Think](../01%20-%20Foundation/02%20-%20How%20Software%20Architects%20Think.md)
- [Engineering Decision Making](../01%20-%20Foundation/03%20-%20Engineering%20Decision%20Making.md)
- [Architecture Principles](../01%20-%20Foundation/08%20-%20Architecture%20Principles.md)

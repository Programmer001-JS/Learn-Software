# Architecture Principles

> **In one line —** a small set of rules that keep a system understandable as it grows, each one bought at a price.

| | |
|---|---|
| **Category** | Foundation Concept |
| **Prerequisites** | [What Is Software Architecture](01%20-%20What%20Is%20Software%20Architecture.md) |
| **Related notes** | [SOLID Principles](../07%20-%20Backend%20Design%20Patterns/SOLID%20Principles.md) · [Clean Architecture](../07%20-%20Backend%20Design%20Patterns/Clean%20Architecture.md) · [Design Patterns](../07%20-%20Backend%20Design%20Patterns/Design%20Patterns.md) |

---

## 1. What principles are for

Principles are **defaults**, not laws. They exist so that a team makes consistent decisions without debating from first principles every time. Every one of them can be broken — but breaking one should be a conscious decision with a reason attached.

---

## 2. Separation of Concerns

Each part of the system should have **one reason to exist**. Business logic does not know about HTTP. The database layer does not know about the user interface.

```text
Controller   knows about HTTP        (request, response, status codes)
    ↓
Service      knows about the domain  (rules, calculations, workflows)
    ↓
Repository   knows about storage     (SQL, tables, transactions)
```

**Bought:** each layer can change independently.
**Paid:** more files, more indirection, more code to read for a simple change.

---

## 3. Single Source of Truth

Every piece of data has **exactly one place** where it is authoritative. Copies are allowed only if they are clearly derived and can be rebuilt.

> [!CAUTION]
> Two systems that both believe they own the same data will eventually disagree — and reconciling them afterwards is one of the most expensive kinds of work there is.

---

## 4. Loose Coupling, High Cohesion

- **Coupling** — how much one component must know about another. Less is better.
- **Cohesion** — how strongly the things inside a component belong together. More is better.

```text
BAD                                  GOOD

[A] ←→ [B]                           [A] → [ interface ] → [B]
 ↕  ╳   ↕                            [C] ──────┘
[C] ←→ [D]                           
everything knows everything          each knows only the contract
```

---

## 5. Explicit over Implicit

Magic saves typing and costs debugging. Configuration that comes from three places with unclear precedence, or a framework that auto-wires things invisibly, is fast to write and painful to diagnose.

---

## 6. Fail Fast and Loudly

Validate input at the boundary and reject it immediately, with a clear error. A system that silently swallows a bad value produces corrupted data later, in a place with no connection to the cause.

```text
Bad input
    ↓
Reject at the edge  →  clear error, nothing corrupted
    ↓            ✕
Accept quietly      →  corrupt row discovered three weeks later
```

---

## 7. Design for Failure

Assume every dependency will be unavailable or slow at some point. Every remote call needs a **timeout**, a defined **retry** policy, and a defined behaviour when it ultimately fails.

> [!IMPORTANT]
> Slow is worse than down. A dependency that is down fails fast; a dependency that takes 30 seconds holds every caller's threads until the whole system stops.

---

## 8. Statelessness where possible

If a server holds no per-user state in memory, any server can handle any request — which is what makes horizontal scaling and simple restarts possible. Push state into a shared store ([Redis](../08%20-%20Databases%20and%20Data/Redis.md), the database) or into the token itself.

```text
Stateful                     Stateless
user must hit server 2       any server works
    ↓                            ↓
scaling and restarts hurt    add or kill servers freely
```

---

## 9. Idempotency

The same operation applied twice should have the same effect as applying it once. Networks retry, users double-click, queues redeliver — an operation that cannot tolerate repetition will eventually be repeated.

---

## 10. Keep It Simple / YAGNI

> [!TIP]
> **YAGNI — "You Aren't Gonna Need It".** Do not build for a requirement you have imagined. The cost is paid now and with certainty; the benefit is speculative.

Complexity is a permanent tax: it is paid at every future change, by every future developer.

---

## 11. Don't Repeat Yourself — carefully

DRY applies to **knowledge**, not to text. Two pieces of code that look identical but change for different reasons should stay separate; merging them creates a coupling that will hurt later.

> [!WARNING]
> Premature abstraction is more expensive than duplication. Duplication is cheap to fix; a wrong abstraction that ten modules depend on is not.

---

## 12. Principle of Least Privilege

Every component, service and user gets the **minimum access** needed to do its job — and nothing more. A service that only reads should not hold write credentials.

---

## 13. The principles in one picture

```text
                    ┌─────────────────────────┐
  Separation of     │   understandable        │
  Concerns          │        system           │
  Loose Coupling ──►│                         │◄── Design for Failure
  Single Source     │   grows without         │    Least Privilege
  of Truth          │      collapsing         │    Idempotency
  Explicit          │                         │    Statelessness
                    └─────────────────────────┘
                              ▲
                        KISS / YAGNI
                 (keeps the others from becoming ceremony)
```

---

## 14. Mental Model

> [!NOTE]
> **Principles are like traffic rules.**

They exist so that strangers can share a road without negotiating every intersection. An ambulance may break them — deliberately, visibly, and with a reason. Breaking them casually is how accidents happen.

---

## 15. Key Takeaway

> [!IMPORTANT]
> Principles are defaults that keep a system understandable; break them consciously and write down why, never by accident.

---

## 16. Common Mistakes

- **Applying every principle everywhere** — a three-file script does not need clean architecture
- **DRY-ing things that are only accidentally similar**
- **Building abstractions for flexibility nobody asked for**
- **Treating principles as morality** rather than as trade-offs with a price
- **Ignoring them entirely** and rediscovering each one through pain

---

## 17. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 18. Workbook Exercise

- [ ] Find one place in your code where two components know too much about each other, and describe the contract that should sit between them.
- [ ] Find one remote call in your project with no timeout, and decide what should happen when it fails.
- [ ] Find one abstraction you built "for later" that is still unused.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Business requirements
    ↓
ARCHITECTURE PRINCIPLES   ← the defaults that shape every decision below
    ↓
Component design
    ↓
Code
```

## 2. Request Flow

```text
Input       a design choice
    ↓
Processing  check it against the principles; note which one it breaks and why
    ↓
Output      a consistent system, or a documented, deliberate exception
```

## 3. Real-World Usage

**Netflix** designs every service on the assumption that its dependencies will fail, using timeouts, fallbacks and circuit breakers by default. "Design for failure" is not advice there — it is enforced by the platform.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A small set of default rules for structuring systems |
| **Why does it exist?** | So teams decide consistently without re-arguing every choice |
| **Where does it belong?** | Between requirements and concrete design |
| **When should I use it?** | As the default — and break them only deliberately, in writing |

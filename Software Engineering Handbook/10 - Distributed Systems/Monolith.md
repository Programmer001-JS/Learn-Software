# Monolith

> **In one line —** one application, one deployment, one database — the default that most teams should still be choosing, and the one they are most often talked out of.

| | |
|---|---|
| **Category** | Architecture Pattern |
| **Architectural Layer** | System |
| **Related notes** | [Microservices](Microservices.md) · [Modular Monolith](Modular%20Monolith.md) · [Clean Architecture](../07%20-%20Backend%20Design%20Patterns/Clean%20Architecture.md) · [Backend Fundamentals](../06%20-%20Backend%20Architecture/Backend%20Fundamentals.md) |

---

## 1. Short Definition

*What is it?*

A monolith is an application built and deployed as a single unit. All the code lives in one codebase, runs in one process, and typically shares one database.

---

## 2. Purpose

*What is its main purpose?*

To keep everything simple and fast: a function call instead of a network call, a transaction instead of a saga, a stack trace instead of distributed tracing.

---

## 3. Problem

*What engineering problem does it solve?*

The problem a monolith solves is **not having a distributed system**. Every distributed architecture pays a permanent tax in latency, failure modes, operational tooling and debugging difficulty. A monolith declines to pay it.

```text
MONOLITH                              DISTRIBUTED
function call — nanoseconds           network call — milliseconds, can fail
BEGIN...COMMIT                        saga with compensating actions
stack trace                           trace IDs across services
one deployment                        orchestration, service discovery, mesh
run it locally with one command       twelve containers
```

---

## 4. Architecture Position

```text
Load balancer
    ↓
┌─────── ONE APPLICATION ───────┐
│  HTTP layer                   │
│  Business logic               │
│  Data access                  │
└───────────────┬───────────────┘
                ↓
            Database
```

Scaling is horizontal, exactly as with any stateless service: run more copies behind the load balancer.

---

## 5. The misconception worth correcting

> [!IMPORTANT]
> **"Monolith" is not the opposite of "scalable", and it is not the opposite of "well-structured".** It describes the *deployment unit*, nothing else.
>
> - A monolith scales horizontally — [Shopify](https://shopify.engineering/), Stack Overflow and GitHub have all run enormous ones.
> - A monolith can be beautifully modular — see [Modular Monolith](Modular%20Monolith.md).
> - A microservice system can be an unmaintainable mess — a *distributed* one, which is worse.
>
> The word people actually mean when they say "monolith" pejoratively is **big ball of mud**, and that is a structure problem, not a deployment problem.

---

## 6. What it gives you

```text
✓ Transactions        one BEGIN...COMMIT across the whole operation
✓ Refactoring         rename across the entire codebase in one commit
✓ Debugging           one stack trace, one log stream, one process
✓ Local development   run it and it works
✓ Latency             no internal network hops at all
✓ Cost                one deployment target, one runtime
✓ Consistency         no eventual consistency to design around
```

---

## 7. Where it genuinely hurts

```text
✗ Deployment coupling      one team's bug blocks everyone's release
✗ Scaling granularity      scale everything, even to fix one hot path
✗ Technology lock-in       one language and runtime for all of it
✗ Blast radius             a memory leak in one feature takes the whole app down
✗ Build times              a large codebase means slow CI
✗ Onboarding               a new developer faces the whole system
```

> [!IMPORTANT]
> Notice that most of these become painful as the **team** grows, not as the traffic grows. That is the honest signal for when to split — and it is the same conclusion the [Microservices](Microservices.md) note reaches from the other direction.

---

## 8. Real World Example

- **Stack Overflow** served enormous traffic from a small number of servers running a .NET monolith — repeatedly cited as evidence of how far one well-tuned application goes.
- **Shopify** runs one of the largest Rails monoliths in existence, deliberately modularised rather than split.
- **Basecamp / 37signals** have argued publicly and consistently for the majestic monolith.
- **Several companies have consolidated microservices back into monoliths** after finding the operational cost exceeded the benefit at their scale.

---

## 9. Scaling a monolith

```text
1. Vertical      a bigger machine — surprisingly effective, and boring
2. Horizontal    more copies behind a load balancer (requires statelessness)
3. Caching       remove most reads from the database
4. Read replicas separate read traffic
5. Extract       pull out ONE genuinely problematic component
```

> [!TIP]
> **Step 5 is not "adopt microservices".** Extracting one component — the video encoder, the search index — because it has a genuinely different scaling profile is a targeted decision. That is very different from splitting everything.

---

## 10. When To Use / When NOT To Use

> [!TIP]
> **Start here. Almost always.** For a new product, a small team, or an unclear domain, a monolith is the correct default — and a [modular monolith](Modular%20Monolith.md) keeps the option to split later.

> [!CAUTION]
> Move away from it when:
> - **Multiple teams** are genuinely blocked by shared deployments
> - **One component** has a wildly different scaling or resource profile
> - **A compliance boundary** requires physical isolation
>
> Not because of traffic. Not because of fashion. Not because of a conference talk about a company with two thousand engineers.

---

## 11. Advantages and Disadvantages

**Advantages**
- Simple to build, run, test and reason about
- ACID transactions across the whole domain
- Fast — no network hops internally
- Cheap to operate
- Straightforward debugging and local development
- Refactoring is a single commit

**Disadvantages**
- Deployment coupling across teams
- Coarse scaling granularity
- One technology stack
- A single failure can affect everything
- Slower builds as the codebase grows
- **Tends toward a big ball of mud without deliberate structure**

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Latency** | Best possible — internal calls are function calls |
| **Throughput** | Scales horizontally like any stateless application |
| **Resource use** | Efficient — one runtime, one process per instance |
| **Startup** | Slower for large applications; relevant for autoscaling |
| **Database** | Usually the real bottleneck, and it would be either way |

> [!IMPORTANT]
> When a monolith is slow, the cause is almost always a **missing index or an N+1 query** — not the architecture. Splitting into services fixes neither, and adds network latency on top.

---

## 13. Security Considerations

- **A single blast radius** — one vulnerability exposes everything in the process, including data other features own
- **Least privilege is harder** — the whole application typically shares one database user with broad rights
- **But**: one deployment means **one place to patch**, one dependency tree to scan, one configuration to secure. Microservices multiply all of those by the number of services
- **Internal module boundaries are not security boundaries** — a compromised code path can call anything in the process, which is precisely the case for keeping compliance-sensitive components separate

---

## 14. Mental Model

> [!NOTE]
> **A monolith is a house; microservices are a street of houses.**
>
> One house is simpler in every respect: one set of pipes, one front door, one place to look when something breaks. It works for one family and for quite a large one. It stops working when several independent families need to renovate on different schedules — which is an occupancy problem, not a plumbing problem.

---

## 15. Mini Architecture Diagram

```text
Load balancer
    ↓
App instance · App instance · App instance   ← horizontal scaling
    ↓
Cache (Redis)
    ↓
Database primary → read replicas
```

---

## 16. Complete Request Flow

```text
POST /orders
    ↓
Load balancer → any instance (stateless, so any will do)
    ↓
Middleware: auth, validation
    ↓
Order service (a MODULE, not a network call)
    ↓
BEGIN TRANSACTION
    inventory check   ← a function call, in the same transaction
    order saved
    stock decremented
COMMIT                ← atomic across the whole operation
    ↓
Email enqueued to a background worker
    ↓
201 returned in ~40 ms
    ↓
An error anywhere → one stack trace, one log stream, one place to look
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> A monolith is a deployment choice, not a quality judgement — it keeps transactions, debugging and latency simple, and you should leave it for organisational reasons rather than technical ones.

---

## 18. Common Mistakes

- **Assuming a monolith cannot scale** — it scales horizontally like anything else
- **Confusing "monolith" with "unstructured"** — that is a big ball of mud
- **Splitting for scale** when the problem is a missing index
- **Never modularising** — internal structure is what keeps it maintainable
- **Stateful instances**, which prevent horizontal scaling
- **Splitting before the domain is understood**, then discovering the boundaries were wrong

---

## 19. Open Source Technologies

- Any backend framework — **Django**, **Rails**, **Spring Boot**, **Laravel**, **ASP.NET Core** — is designed for this shape
- **Docker** — a monolith containerises perfectly well
- **Modular monolith tooling**: `import-linter`, **ArchUnit**, **Packwerk** (Rails)
- **Load balancers**: Nginx, HAProxy, cloud load balancers

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] If you believe your monolith cannot scale, measure where the time actually goes before accepting that.
- [ ] Identify one component with a genuinely different scaling profile — is there one?
- [ ] Check whether your application is stateless enough to run five copies behind a load balancer today.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Load balancer → N identical app instances → cache → database
```

## 2. Request Flow

```text
Input       a request to any instance
    ↓
Processing  handled in-process with function calls and one transaction
    ↓
Output      a fast response, with one stack trace when anything fails
```

## 3. Real-World Usage

**Shopify** runs one of the largest Rails monoliths in the world and chose to modularise it rather than split it. It is the strongest available counter-argument to the idea that scale forces a distributed architecture.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An application built and deployed as a single unit |
| **Why does it exist?** | Because distribution has a permanent cost worth avoiding |
| **Where does it belong?** | Behind a load balancer, like any stateless application |
| **When should I use it?** | By default — leaving it for organisational reasons, not traffic |

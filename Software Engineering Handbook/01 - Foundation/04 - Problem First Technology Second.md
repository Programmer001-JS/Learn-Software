# Problem First, Technology Second

> **In one line —** never start with "I want to use X"; start with "what breaks if I do nothing?"

| | |
|---|---|
| **Category** | Foundation Principle |
| **Prerequisites** | [Engineering Decision Making](03%20-%20Engineering%20Decision%20Making.md) |
| **Related notes** | [Learning Philosophy](../00%20-%20Introduction/Learning%20Philosophy.md) · [Architecture Principles](08%20-%20Architecture%20Principles.md) |

---

## 1. The principle

Technology is an **answer**. If you cannot state the question, you cannot evaluate the answer — and you certainly cannot compare it to alternatives.

```text
WRONG                          RIGHT

I want to learn Kafka          Producers are faster than consumers
    ↓                              ↓
Where can I use Kafka?         What solves that?
    ↓                              ↓
Force it into the project      Queue → Kafka, RabbitMQ, Redis Streams, SQS
                                   ↓
                               Compare, choose, know why
```

---

## 2. Why this order matters

When you start from the technology, three things go wrong:

- You cannot recognise the **alternatives**, because you never named the problem they all solve.
- You cannot tell when the tool is **wrong**, so you use it everywhere.
- You add **permanent operational cost** for a problem you may not even have.

> [!CAUTION]
> "We should use microservices" is not a plan. It is a solution looking for a problem — and if the actual problem is a slow database query, microservices will make it worse.

---

## 3. The three questions before any technology

1. **What exactly is the problem?** — in one sentence, with a number in it if possible
2. **What happens if I do nothing?** — if the answer is "nothing much", stop here
3. **What is the simplest thing that solves it?** — start there, not at the sophisticated end

---

## 4. Problems and their answer families

Learning the **left column** is what makes you an engineer. The right column changes every few years.

| Problem | Family of answers |
|---|---|
| The same data is read repeatedly | Cache — [Redis](../08%20-%20Databases%20and%20Data/Redis.md), Memcached, CDN, local memory |
| Work is slower than the request can wait | Queue + worker — RabbitMQ, Kafka, Celery, SQS |
| One server cannot handle the traffic | Horizontal scaling + [load balancer](../14%20-%20Scalability%20and%20Reliability/Load%20Balancing.md) |
| Related data must stay consistent | Relational database + transactions |
| Search across text | Search engine — Elasticsearch, Meilisearch, Postgres full-text |
| Find similar meaning, not exact words | Vector database — Qdrant, pgvector, Weaviate |
| Deploys break because environments differ | Containers — [Docker](../13%20-%20DevOps%20and%20Delivery/Docker.md) |
| Identity must be checked on every request | Sessions or [JWT](../09%20-%20Security/JWT.md) |
| Files are too big for the database | Object storage — S3, MinIO |

> [!TIP]
> Memorise this table by **problem**, not by tool. When a new tool appears, you will immediately know which row it belongs to — and therefore what it competes with.

---

## 5. Architecture Position

This principle sits at the very top of every decision, above architecture itself.

```text
Business problem
    ↓
Engineering problem      ← "the same query runs 10,000 times a minute"
    ↓
Family of solutions      ← "this is a caching problem"
    ↓
Specific technology      ← "Redis, because we already run it"
    ↓
Implementation
```

Skipping the middle two steps is how projects acquire technology nobody can justify.

---

## 6. Real World Example

- **Instagram** stayed on plain Django and PostgreSQL far longer than fashionable advice suggested, because their real problem was image delivery — solved with a CDN and object storage, not by rewriting the application.
- **Stack Overflow** ran an enormous site on a small number of servers with heavy caching, because their problem was read volume of mostly-static pages, not write throughput.

Both are examples of matching the solution to the actual problem rather than to the industry trend.

---

## 7. The "resume-driven development" trap

> [!WARNING]
> Choosing technology because it is impressive rather than appropriate produces systems that are hard to run, hard to hire for, and hard to fix at 3am. The cost is paid by whoever maintains it — often you, six months later.

Learning a new technology is a perfectly good goal. Do it in a side project where the cost of being wrong is zero, not in a system people depend on.

---

## 8. Mental Model

> [!NOTE]
> **Technology is a medicine. Prescribing before diagnosis is malpractice.**

Every medicine has side effects. A doctor who prescribes the newest drug to every patient because they read about it is not being modern — they are being careless.

---

## 9. Key Takeaway

> [!IMPORTANT]
> Understand the problem so precisely that the technology becomes obvious — and if it does not become obvious, you have not understood the problem yet.

---

## 10. Common Mistakes

- Starting a project by choosing the stack
- Adopting a tool because a large company uses it, without checking whether you share their constraints
- Solving a problem you do not yet have
- Ignoring the option of solving it with code you already own
- Confusing "I want to learn this" with "this project needs this"

---

## 11. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 12. Workbook Exercise

- [ ] Take every technology in your current project and write the one-sentence problem it solves. Anything without an answer is a candidate for removal.
- [ ] Pick a technology you want to learn and find the real problem it answers, then name two competing answers to the same problem.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Problem
    ↓
Family of solutions
    ↓
Specific technology
    ↓
Implementation
```

## 2. Request Flow

```text
Input       "the app is slow"
    ↓
Processing  measure → locate → name the engineering problem → list the family
    ↓
Output      a justified technology choice, with alternatives understood
```

## 3. Real-World Usage

**Instagram** scaled to tens of millions of users on a deliberately plain Django + PostgreSQL stack, because their bottleneck was media delivery rather than application logic. Solving the real problem kept their architecture small.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The rule that the problem is defined before the tool is chosen |
| **Why does it exist?** | Because tool-first thinking hides alternatives and adds unjustified cost |
| **Where does it belong?** | At the top of every technical decision |
| **When should I use it?** | Every single time you are tempted to add something to the stack |

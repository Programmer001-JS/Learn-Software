# Learning Philosophy

> **In one line —** understand the problem before the tool, the shape before the detail, and the trade-off before the answer.

---

## Why this handbook is built the way it is

Most technical learning fails in the same way: you learn **what** a tool does, never **why it exists**, and so you cannot tell when it is the wrong choice. Six months later the tool has changed its API and your knowledge is worthless, because what you memorised was syntax rather than reasoning.

This handbook is organised around the opposite bet: **the reasoning outlives the tool.**

```text
Syntax          →  obsolete in 2 years
API details     →  obsolete in 3 years
The problem it solves  →  still true in 20 years
```

---

## Principle 1 — Problem before technology

Never start with "I want to learn Kafka." Start with "what happens when one service produces work faster than another can consume it?" Kafka then becomes an *answer* rather than a *topic*, and you automatically understand its alternatives, because they are answers to the same question.

See: [Problem First Technology Second](../01%20-%20Foundation/04%20-%20Problem%20First%20Technology%20Second.md)

---

## Principle 2 — Position before implementation

Before learning how something works, learn **where it sits**. A concept you cannot place in a diagram is a concept you do not understand.

```text
Browser  →  Frontend  →  API  →  Cache  →  Database
```

Every technology in this handbook has a place in a picture like this. That is why every note contains a diagram — the diagram is not decoration, it is the actual lesson.

---

## Principle 3 — Breadth first, depth on demand

> [!IMPORTANT]
> It is far more useful to know 200 concepts at a shallow level than 5 concepts deeply — **as long as you know which 5 to go deep on when the time comes.**

Shallow knowledge across the whole map lets you:

- Recognise which part of the system a problem lives in
- Ask a precise question instead of a vague one
- Evaluate a proposed solution without having built it before

Depth is expensive, so buy it only where you are actually working.

---

## Principle 4 — Trade-offs, never verdicts

There is no best database, best language or best architecture. Every choice buys something and pays for it somewhere else.

| Instead of asking | Ask |
|---|---|
| "Is MongoDB good?" | "What does MongoDB give up to get schema flexibility?" |
| "Should I use microservices?" | "What does splitting cost me in latency and operations?" |
| "Is Redis fast?" | "Fast at what, and at what cost in memory and consistency?" |

> [!TIP]
> When someone answers a technology question without mentioning a downside, they are selling, not explaining.

---

## Principle 5 — Explain it out loud

The strongest test of understanding is not a quiz, it is a sentence. If you cannot explain a concept to someone non-technical in two sentences using a real-world comparison, you have memorised it rather than understood it.

That is why every note has a **Mental Model** section. Redis is a refrigerator. A queue is a waiting line. Docker is a shipping container. These comparisons are imperfect on purpose — an imperfect model you can reason with beats a perfect definition you cannot.

---

## Principle 6 — Learning is not reading

```text
Read                →  20% understood, 5% retained
Read + draw         →  50% understood
Read + draw + explain  →  80% understood
Read + draw + explain + build  →  actually learned
```

> [!CAUTION]
> Reading feels like progress because it is comfortable. Building feels like failure because it is uncomfortable. The uncomfortable one is the one that works.

---

## Principle 7 — Write down what confused you

The single most valuable thing in this handbook after a year will not be my text. It will be your **Personal Notes** sections — the record of what confused you and how you resolved it. Confusion is a signal that you have found the edge of your understanding, and the edge is the only place where learning happens.

Keep those notes in section 25 of each lesson, and the bigger ones in [17 - Personal Knowledge Base](../17%20-%20Personal%20Knowledge%20Base).

---

## The goal

> [!IMPORTANT]
> The goal is not to become someone who knows every technology. It is to become someone who, when facing an unfamiliar technology, already knows **which questions to ask** — and can get to a competent decision in an hour.

That skill is what separates a developer from an architect, and it is entirely learnable.

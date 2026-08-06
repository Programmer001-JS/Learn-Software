# Software Engineering Handbook

> **A structured path from "how does a computer actually work" to "how would I design Netflix".**

This handbook is a personal knowledge base built around one idea: **every concept in software engineering can be studied with the same set of questions.** Learn the questions once, and every new technology becomes far less intimidating.

---

## What this handbook is

- A **map**, not an encyclopedia. Each note tells you what a thing is, why it exists, and where it sits in an architecture.
- **Concise by design.** Two to four sentences per question. Depth is added later, when you actually use the technology.
- **Architecture-first.** Every note contains a diagram showing where the concept belongs in a real system.
- **Yours to extend.** Sections 25 and 26 of every lesson are deliberately empty — they are your notes and your exercise.

---

## What this handbook is not

> [!WARNING]
> This is **not** a substitute for building things. Reading 200 notes without writing code produces the illusion of knowledge, not knowledge. Every lesson ends with an exercise for exactly this reason.

It is also not a tutorial. You will not find installation steps or API references here — those live in official documentation and go out of date. What is here is the part that does *not* go out of date: **the reasoning.**

---

## Table of Contents

| # | Section | What you learn |
|---|---|---|
| **00** | [Introduction](.) | How to use this handbook |
| **01** | [Foundation](../01%20-%20Foundation) | How engineers and architects think |
| **02** | [Computer Science Fundamentals](../02%20-%20Computer%20Science%20Fundamentals) | Hardware, operating systems, processes |
| **03** | [Programming Languages and Runtime](../03%20-%20Programming%20Languages%20and%20Runtime) | How your code actually runs |
| **04** | [Networking and Internet](../04%20-%20Networking%20and%20Internet) | How machines talk to each other |
| **05** | [Frontend Architecture](../05%20-%20Frontend%20Architecture) | The browser and everything in it |
| **06** | [Backend Architecture](../06%20-%20Backend%20Architecture) | Servers, runtimes, frameworks |
| **07** | [Backend Design Patterns](../07%20-%20Backend%20Design%20Patterns) | How to structure code that lasts |
| **08** | [Databases and Data](../08%20-%20Databases%20and%20Data) | Where the truth is stored |
| **09** | [Security](../09%20-%20Security) | Identity, secrets, and attacks |
| **10** | [Distributed Systems](../10%20-%20Distributed%20Systems) | Many machines, one system |
| **11** | [AI Engineering](../11%20-%20AI%20Engineering) | Models, embeddings, RAG, agents |
| **12** | [Cloud Architecture](../12%20-%20Cloud%20Architecture) | Renting infrastructure instead of owning it |
| **13** | [DevOps and Delivery](../13%20-%20DevOps%20and%20Delivery) | Getting code into production safely |
| **14** | [Scalability and Reliability](../14%20-%20Scalability%20and%20Reliability) | Staying up under load |
| **15** | [Real World System Design](../15%20-%20Real%20World%20System%20Design) | Case studies of real architectures |
| **16** | [Templates](../16%20-%20Templates) | Reusable templates and checklists |
| **17** | [Personal Knowledge Base](../17%20-%20Personal%20Knowledge%20Base) | Your own notes, ideas and questions |

---

## The one page to read first

**[The Complete System Map](The%20Complete%20System%20Map.md)** — the whole handbook as a single journey: from a user's click to the product in their hands, in 24 stations. Each station says what happens there, which concepts live there, and what can go wrong. It contains no definitions — every definition is one click away.

```text
If the table above tells you WHAT is in the handbook,
the map tells you WHERE each thing sits, and what is waiting for whom.
```

It comes in two formats, for two different jobs:

| | For | Shape |
|---|---|---|
| **[The Complete System Map](The%20Complete%20System%20Map.md)** | Looking things up | Vertical, station by station, every concept linked |
| **[The same map, as a Poster](The%20Complete%20System%20Map%20-%20Poster.md)** | The wall | Wide landscape, five posters, built to print on A3 |

> [!TIP]
> Read the poster's first diagram before anything else, and again after every few notes. It is the difference between 204 separate facts and one system.

---

## The 26 questions

Every technology note answers the same list. You can find the full template in **[Universal Technology Learning Template](../16%20-%20Templates/Universal%20Technology%20Learning%20Template.md)**, but the short version is:

```text
What is it?          →  Why does it exist?     →  Where does it live?
    ↓                        ↓                         ↓
How does it work?    →  What are the trade-offs?  →  When do I use it?
```

If you can answer those six, you understand the technology well enough to make a decision about it.

---

## Start here

1. **[The Complete System Map](The%20Complete%20System%20Map.md)** — the whole picture, in one file
2. **[How To Use This Handbook](How%20To%20Use%20This%20Handbook.md)** — the workflow
3. **[Learning Philosophy](Learning%20Philosophy.md)** — why it is built this way
4. **[Architect Mindset](Architect%20Mindset.md)** — the shift from coder to architect
5. **[01 - Foundation](../01%20-%20Foundation/01%20-%20What%20Is%20Software%20Architecture.md)** — the first real lesson

> [!IMPORTANT]
> Do not read this handbook front to back like a novel. Read Foundation, then jump to whatever you are actually working on. The map is there so you never get lost — not so you walk every road.

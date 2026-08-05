# AI Driven Development Workflow

> **In one line —** AI makes writing code cheap, which shifts the bottleneck to knowing what should be written and verifying what was.

| | |
|---|---|
| **Category** | Foundation / Practice |
| **Related notes** | [Documentation Driven Development](07%20-%20Documentation%20Driven%20Development.md) · [Software Development Lifecycle](05%20-%20Software%20Development%20Lifecycle.md) · [AI Agents](../11%20-%20AI%20Engineering/AI%20Agents.md) |

---

## 1. What changed

For fifty years, typing code was the slow part of software engineering, so tools and processes were built around making typing faster. That constraint is largely gone. What has *not* changed is that someone must decide what to build, and someone must be accountable for whether it is correct.

```text
Before                          Now

Idea                            Idea
  ↓  slow                         ↓  slow      ← still the bottleneck
Design                          Design
  ↓  slow                         ↓  fast
Write code                      Generate code
  ↓  slow                         ↓  slow      ← now the bottleneck
Review & verify                 Review & verify
```

> [!IMPORTANT]
> The value of an engineer moves toward the two ends: **defining the problem precisely** and **verifying the result rigorously.** The middle is increasingly automated.

---

## 2. What AI is genuinely good at

- Boilerplate, glue code, and translations between formats
- Explaining unfamiliar code or an unfamiliar error
- Producing a first draft of tests, docs, migrations, configs
- Exploring an approach quickly so you can reject it cheaply
- Answering "which family of solutions does this problem belong to?"

## 3. What it is not reliable at

> [!CAUTION]
> AI is confidently wrong in ways that look exactly like being right. The failure mode is not a syntax error you can see — it is a plausible design that is subtly unsuitable for your constraints.

- Knowing **your** constraints, traffic, budget and team — unless you tell it
- Architectural decisions with long-term consequences
- Anything security-critical without a human review
- Facts about very recent library versions and APIs
- Knowing when the honest answer is "you do not need this at all"

---

## 4. A working loop

```text
1. Describe the problem, constraints and acceptance criteria in writing
    ↓
2. Ask for the DESIGN first, not the code
    ↓
3. Challenge it — "what breaks?", "what did you assume?", "what is the simpler option?"
    ↓
4. Generate the code in small, reviewable pieces
    ↓
5. Read every line. If you cannot explain it, do not merge it.
    ↓
6. Run it. Test it. Especially the failure paths.
    ↓
7. Record the reasoning (ADR / commit message), not just the result
```

> [!TIP]
> Step 2 is what separates useful AI work from expensive mess. Asking for the design first turns the model into a thinking partner rather than a very fast intern with no context.

---

## 5. Context is the whole game

An AI's output quality is bounded by what it knows about your system. The same question with and without context produces completely different answers.

| Weak prompt | Strong prompt |
|---|---|
| "Add caching" | "This endpoint runs the same query ~2000×/min, data changes hourly, we already run Redis, staleness up to 5 min is fine. Propose two options and their trade-offs." |

This is why [Documentation Driven Development](07%20-%20Documentation%20Driven%20Development.md) matters more now, not less: written architecture, ADRs and clear README files are the context that makes AI useful.

---

## 6. Architecture Position

AI sits **beside** the engineer at several stages of the lifecycle — it does not replace a stage.

```text
Requirements  ← AI helps clarify and challenge
    ↓
Design        ← AI proposes options; the engineer decides
    ↓
Implementation← AI generates; the engineer reviews
    ↓
Testing       ← AI drafts tests; the engineer defines what "correct" means
    ↓
Operation     ← AI helps debug; the engineer is accountable
```

---

## 7. The verification problem

Generating 500 lines takes seconds. Verifying 500 lines takes as long as it always did. Teams that only optimise generation end up with a growing pile of code nobody has actually read.

> [!WARNING]
> Never merge code you cannot explain. "The AI wrote it" is not an answer during an incident at 3am, and it is not an answer in a code review either.

Practical defences:

- Small changes, reviewed individually
- Tests you wrote or at least fully understood
- Static analysis and type checking
- A security review for anything touching auth, input handling or secrets

---

## 8. Skills that become more valuable

- **Precise problem definition** — vague requests produce vague systems
- **Reading code fast and critically**
- **Testing and verification**
- **Architecture and trade-off reasoning** — the part with real consequences
- **Debugging** — understanding a system you did not type yourself

## 9. Skills that become less valuable

- Memorising syntax and API signatures
- Writing boilerplate by hand
- Being the person who knows one framework very well

---

## 10. Mental Model

> [!NOTE]
> **AI is a very fast junior engineer with an enormous library and no memory of your project.**

It will produce a great deal of plausible work extremely quickly. It has no idea what your deadline is, what broke last month, or why that ugly workaround exists. You are the senior engineer in this pairing, and the accountability is entirely yours.

---

## 11. Key Takeaway

> [!IMPORTANT]
> AI removes the cost of writing code, not the cost of being wrong — so invest the time you save into problem definition and verification.

---

## 12. Common Mistakes

- **Asking for code before agreeing on the design**
- **Merging code you have not read**
- **Giving no context**, then blaming the model for a generic answer
- **Trusting confident answers about recent library versions** without checking the docs
- **Skipping security review** because the code "looks professional"
- **Letting AI make architectural decisions** — those need your constraints and your accountability

---

## 13. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 14. Workbook Exercise

- [ ] Take a task you would normally start coding immediately, and instead write the problem, constraints and acceptance criteria first. Compare the result.
- [ ] Take a piece of AI-generated code you have used and explain every line out loud. Note where you could not.
- [ ] Write the context paragraph about your project that you would paste at the start of any AI session.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Engineer
    ↓  problem + constraints + context
AI assistant
    ↓  design options
Engineer decides
    ↓
AI generates code
    ↓
Engineer reviews, tests, is accountable
    ↓
Production
```

## 2. Request Flow

```text
Input       a precisely described problem with constraints
    ↓
Processing  AI proposes design and code; engineer challenges and verifies
    ↓
Output      reviewed, tested code plus a written record of the reasoning
```

## 3. Real-World Usage

Engineering teams across the industry now use AI assistants inside the editor and the pull-request flow. The teams that benefit are the ones that kept code review, testing and architectural ownership in human hands — the generation step got faster, the accountability did not move.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A workflow where AI handles generation and the engineer handles definition and verification |
| **Why does it exist?** | Because writing code stopped being the bottleneck |
| **Where does it belong?** | Alongside the engineer at every lifecycle stage |
| **When should I use it?** | Constantly for drafting and exploring — never as the final authority on a decision |

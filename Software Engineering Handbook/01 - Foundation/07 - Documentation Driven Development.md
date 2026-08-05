# Documentation Driven Development

> **In one line —** write the explanation before the code; if the explanation is hard to write, the design is wrong.

| | |
|---|---|
| **Category** | Foundation / Practice |
| **Abbreviation** | DDD *(not to be confused with [Domain Driven Design](../07%20-%20Backend%20Design%20Patterns/Domain%20Driven%20Design.md))* |
| **Related notes** | [Architecture Decision Record](../16%20-%20Templates/Architecture%20Decision%20Record.md) · [AI Driven Development Workflow](06%20-%20AI%20Driven%20Development%20Workflow.md) |

---

## 1. Short Definition

Documentation Driven Development means writing the **documentation first** — what the thing does, what it accepts, what it returns, why it exists — and only then implementing it. The document is the design; the code is the consequence.

---

## 2. The problem it solves

Code answers *what* the system does. It can never answer:

- **Why** was it built this way?
- Which alternatives were considered and rejected?
- Which constraint forced this ugly workaround?
- What is this component **not** supposed to do?

That knowledge exists only in people's heads, and it leaves the company with them. Six months later nobody dares to touch the code, because nobody knows which parts of it are load-bearing.

---

## 3. The mechanism

```text
Write the doc
    ↓
Reading it back reveals the confusion   ← this is the actual value
    ↓
Fix the design on paper (cheap)
    ↓
Implement
    ↓
Update the doc when reality disagrees
```

> [!IMPORTANT]
> Writing forces precision that thinking does not. A design that feels clear in your head very often falls apart in the second paragraph — and finding that out costs a paragraph instead of a sprint.

---

## 4. What to write, and when

| Artifact | Written when | Answers |
|---|---|---|
| **README** | At project start | What is this, how do I run it? |
| **ADR** | Before an irreversible decision | Why did we choose this? |
| **API contract** | Before implementing an endpoint | What goes in, what comes out, what can fail? |
| **Architecture diagram** | Before adding a component | Where does this fit? |
| **Runbook** | Before going to production | What do I do when it breaks at 3am? |
| **Changelog** | At release | What changed for the user? |

---

## 5. API-first as the clearest example

```text
1. Write the endpoint contract        POST /orders → 201 { id } | 400 | 409
    ↓
2. Frontend and backend agree on it
    ↓
3. Both build in parallel against the contract
    ↓
4. They integrate without surprises
```

Without this, the frontend guesses, the backend improvises, and integration week becomes a negotiation. Tools like OpenAPI turn the document into something machine-checkable.

---

## 6. Architecture Position

Documentation sits **before and beside** every stage of the lifecycle, not at the end of it.

```text
Requirements  →  spec / problem statement
    ↓
Design        →  ADR + architecture diagram
    ↓
Interfaces    →  API contract
    ↓
Implementation→  code + inline "why" comments
    ↓
Operation     →  runbook + monitoring notes
```

---

## 7. Why this matters more in the AI era

> [!TIP]
> Written architecture, ADRs and clear contracts are exactly the **context** an AI assistant needs to produce useful output. A well-documented project makes AI dramatically more effective; an undocumented one makes it confidently generic.

---

## 8. Documentation that is worth writing

- **Why**, not what — the code already shows what
- **Contracts** — inputs, outputs, errors
- **Constraints and assumptions** — "this assumes fewer than 1000 items"
- **Failure behaviour** — what happens when the dependency is down
- **Rejected alternatives** — so nobody re-litigates the decision annually

## 9. Documentation that rots and hurts

> [!CAUTION]
> Wrong documentation is worse than none, because people trust it. Anything that duplicates the code line by line will drift out of date within weeks.

- Comments restating the obvious (`i = i + 1  // increment i`)
- Step-by-step tutorials of a UI that changes every sprint
- Auto-generated docs nobody reads and nobody maintains

---

## 10. Mental Model

> [!NOTE]
> **Documentation = the blueprint, not the photograph of the finished building.**

A blueprint is drawn before construction and is what construction follows. A photograph is taken afterwards and describes what happened to get built. Most teams take photographs and call it documentation.

---

## 11. Key Takeaway

> [!IMPORTANT]
> If you cannot explain the design clearly in writing, you do not yet understand it well enough to build it.

---

## 12. Common Mistakes

- **Writing documentation after shipping** — by then the reasoning is already forgotten
- **Documenting *what* instead of *why***
- **Never updating it**, so it becomes actively misleading
- **Documenting everything**, so nobody reads any of it
- **Keeping it far from the code**, in a wiki nobody opens — keep it in the repository

---

## 13. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 14. Workbook Exercise

- [ ] Take the next feature you plan to build and write its README section *before* writing any code.
- [ ] Write one ADR for a decision you already made, from memory. Notice what you can no longer reconstruct.
- [ ] Write the runbook entry for the most likely failure of your current project.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Idea
    ↓
Written spec / contract / ADR
    ↓
Implementation
    ↓
Documentation updated with reality
```

## 2. Request Flow

```text
Input       an idea or a decision to make
    ↓
Processing  write it down; the writing exposes the gaps
    ↓
Output      a design that survived contact with a blank page
```

## 3. Real-World Usage

**Amazon** replaced slide decks with six-page written narratives, read in silence at the start of a meeting. The rule exists because writing a coherent narrative exposes weak reasoning that bullet points hide.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Writing the design and contracts before implementing them |
| **Why does it exist?** | Because the "why" behind code cannot be recovered from the code |
| **Where does it belong?** | Before and beside every lifecycle stage, stored in the repository |
| **When should I use it?** | For anything more than a throwaway script |

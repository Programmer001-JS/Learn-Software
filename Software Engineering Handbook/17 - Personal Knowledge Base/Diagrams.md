# Diagrams

> **In one line —** your own drawings of systems you actually work on; drawing one from memory is the fastest way to discover what you do not understand.

| | |
|---|---|
| **Category** | Personal *(yours to fill — this file is scaffolding)* |
| **Related notes** | [Technology Notes](Technology%20Notes.md) · [Questions](Questions.md) · [Universal Technology Learning Template](../16%20-%20Templates/Universal%20Technology%20Learning%20Template.md) · [Architect Mindset](../00%20-%20Introduction/Architect%20Mindset.md) |

---

## Why draw it yourself

```text
Reading a diagram          → recognition. Feels like understanding.
Drawing it from memory     → recall. Reveals every gap immediately.
```

> [!IMPORTANT]
> **The gaps are the point.** Draw your own system from memory and you will stop somewhere specific — you will not know how a request reaches the worker tier, or where TLS terminates, or which component holds the session. That stopping point is more valuable than the finished diagram, and it belongs straight into [Questions](Questions.md).

> [!TIP]
> **Draw before you read the code, then check.** Predicting the shape and then verifying it forms far stronger memory than tracing the code first — and the places where you predicted wrongly are the places where your mental model has been quietly misleading you.

---

## Why plain text

```text
ASCII in a Markdown file
    ✓ diffs in Git, so changes are visible
    ✓ readable everywhere, forever, with no tool
    ✓ fast to edit — which means it stays UP TO DATE
    ✗ ugly

A drawing tool
    ✓ pretty
    ✗ a binary file nobody can diff
    ✗ out of date within a month, because editing it is a chore
```

> [!IMPORTANT]
> **An ugly diagram that is correct beats a beautiful one that is six months stale.** The reason architecture diagrams are almost always wrong is that updating them is friction; text has almost none. If a diagram needs to be presentable, generate that version from this one — do not replace this one.

---

## The four diagrams worth having

```text
1. REQUEST PATH        what happens to one user request, end to end
2. DATA STORES         what holds state, and which is the source of truth
3. TRUST BOUNDARIES    where untrusted input enters, where credentials are used
4. FAILURE DOMAINS     what exists only ONCE, and what dies together
```

> [!TIP]
> **Number 4 is the one nobody draws, and it is the one that changes decisions.** Take the request-path diagram and circle everything that exists exactly once. That is your availability work, in priority order, and it is usually shorter and more mundane than expected — one cache node, one NAT gateway, one certificate renewed by hand.

---

## Symbols worth keeping consistent

```text
    ↓  →         flow direction
    ═══          the boundary between synchronous and asynchronous
    ┌──┐         a component you own
    ( )          a component someone else operates
    ⚠            a single point of failure
    ◄── label    the interesting constraint, annotated
```

> [!TIP]
> **Annotate the constraint, not the component.** "DB primary" tells a reader nothing they could not guess; "DB primary ◄── all writes; the real ceiling" tells them what the diagram is *for*. The best diagrams in this handbook are the ones where the labels carry the argument.

---

## My systems

*One section per system. Update the diagram in the same pull request that changes the system, or it will drift.*

```text
## <system name>          LAST UPDATED  YYYY-MM-DD

<the diagram>

NOTES
    <the constraint a newcomer would not guess>
    <what exists only once>
    <what I am unsure about>   → and put that in Questions.md
```

<!--
Example of the shape, to delete once you have real entries:

## Main application         LAST UPDATED  2026-05-02

                internet
                    │
            ( CDN / edge )          ← static + rate limiting
                    │
            ┌───────▼────────┐
            │ load balancer  │      AZ-a + AZ-b
            └───────┬────────┘
        ┌───────────┼───────────┐
    ┌───▼───┐   ┌───▼───┐   ┌───▼───┐
    │  app  │   │  app  │   │  app  │   stateless
    └───┬───┘   └───┬───┘   └───┬───┘
        └─────┬─────┴─────┬─────┘
              ▼           ▼
        ┌──────────┐  ╔═══════════╗
        │ Redis ⚠  │  ║  queue     ║ ══► worker ⚠  (one instance)
        │ sessions │  ╚═══════════╝
        └──────────┘
              │
        ┌─────▼──────────────────┐
        │ DB primary ◄── all writes; the ceiling
        │      │ sync
        │ standby (AZ-b)
        └────────────────────────┘

NOTES
    ⚠ Redis is single-node — sessions die with it; everyone is logged out
    ⚠ one worker instance — no redundancy, and the queue silently backs up
    the TLS certificate on the admin subdomain is still renewed by hand
    NOT SURE where the session TTL is configured  → Questions.md
-->

---

## Diagrams I have drawn from memory

*Track this deliberately. Re-drawing the same system three months later is a genuine test of whether the understanding stuck.*

| System | Date | What I got wrong |
|---|---|---|
| | | |

> [!IMPORTANT]
> **The "what I got wrong" column is the whole exercise.** Not knowing where a component sits is a gap; believing it sits somewhere it does not is a defect in your mental model, and those are what produce confident wrong decisions in design discussions.

---

## Review prompt

- [ ] Which diagram here is out of date? Fix it or date-stamp it as unverified.
- [ ] Redraw one system from memory and compare with the version above.
- [ ] Circle everything that exists exactly once. Is anything on that list surprising?
- [ ] Which diagram do I not have yet — request path, data, trust boundaries, failure domains?
- [ ] Did anything I could not draw make it into [Questions](Questions.md)?

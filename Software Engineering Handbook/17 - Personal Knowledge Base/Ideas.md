# Ideas

> **In one line —** somewhere to put ideas so they stop occupying your attention; capture is cheap, and most of them should die here rather than in a repository.

| | |
|---|---|
| **Category** | Personal *(yours to fill — this file is scaffolding)* |
| **Related notes** | [Questions](Questions.md) · [Technology Notes](Technology%20Notes.md) · [Task Template](../16%20-%20Templates/Task%20Template.md) · [Architecture Decision Record](../16%20-%20Templates/Architecture%20Decision%20Record.md) |

---

## The point of this file

```text
An uncaptured idea does not sit still
    it interrupts you while you work
    it returns at 2 a.m.
    it gets half-started in a branch and abandoned
    ↓
Written down, it stops demanding attention
and can be judged later, calmly
```

> [!IMPORTANT]
> **Capture is not commitment.** The value of this file comes from writing ideas down *without* deciding anything about them. Most ideas look worse in a month, which is exactly what you want to find out before spending a weekend on one. A list that only contains ideas you intend to build is a backlog, not an ideas file — and a backlog does not free your attention the same way.

> [!TIP]
> **Read the list once a month and delete freely.** An idea you have not wanted to start in three months is not an idea you want; deleting it is information, not failure. The ones that survive several passes are the ones worth a [task](../16%20-%20Templates/Task%20Template.md).

---

## Capture

*Anything, in one line, unjudged. Newest at the top. No structure required.*

- <idea>
- <idea>

<!--
Examples of the shape, to delete once you have real entries:

- a CLI that reads pg_stat_statements and prints the three queries worth fixing
- a script that fails CI if any external call in the diff has no timeout
- write up the incident from March as a blameless review, properly
- try running the whole stack on ARM and measure the cost difference
- a small tool that diffs two Terraform plans for humans
-->

---

## Worth a second look

*Ideas that survived at least one review pass. Add one line on why.*

```text
## <idea>
WHY IT KEEPS COMING BACK   <the problem it would actually solve>
SMALLEST VERSION           <what could be built in an afternoon to test it>
WHAT WOULD MAKE IT WORTH IT <the condition>
```

> [!TIP]
> **Always write the "smallest version" line, because it is what stops an idea becoming a project.** Almost every idea has a version that takes an afternoon and answers the interesting question. If you cannot think of one, the idea is not yet understood well enough to start — which is useful to know before committing a month to it.

---

## Started

*Ideas you acted on, with the outcome. Keep the failures — they are the more useful half.*

| Idea | Started | Outcome | Verdict |
|---|---|---|---|
| | | | |

> [!IMPORTANT]
> **Record the ones that did not work, and why.** Without that, you will have the same idea again in eighteen months and rediscover the same wall. "Tried it; the API rate limit makes it useless" is a permanently valuable sentence.

---

## Deliberately rejected

*Ideas you decided against, with the reason. This section prevents re-litigating the same thing annually.*

- **<idea>** — rejected because <reason>, <date>

---

## Improvements to this handbook

*Notes to write, links to add, sections that are wrong. Keep this separate from product ideas.*

- [ ] <note that is missing>
- [ ] <two notes that should be linked>
- [ ] <something in a note that turned out to be wrong or out of date>

> [!TIP]
> **The most valuable handbook improvement is almost always a link, not a new note.** The handbook's value grows with connections; a note that sits unlinked is a file, while a note reachable from three others is part of a map.

---

## Review prompt

*Once a month:*

- [ ] Delete anything from Capture that no longer interests me. No guilt.
- [ ] Promote at most one or two ideas to "Worth a second look".
- [ ] For each promoted idea, write its smallest testable version.
- [ ] Move anything I actually intend to do into a real task, with acceptance criteria.
- [ ] Record the outcome of anything I started, including the ones that failed.

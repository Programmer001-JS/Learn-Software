# Technology Notes

> **In one line —** your own notes on technologies you have actually used; the handbook has the reasoning, this file has the things you can only learn by doing.

| | |
|---|---|
| **Category** | Personal *(yours to fill — this file is scaffolding)* |
| **Related notes** | [Universal Technology Learning Template](../16%20-%20Templates/Universal%20Technology%20Learning%20Template.md) · [Questions](Questions.md) · [Ideas](Ideas.md) · [How To Use This Handbook](../00%20-%20Introduction/How%20To%20Use%20This%20Handbook.md) |

---

## What belongs here

```text
THIS FILE                              THE HANDBOOK
"the flag that took me two hours       "what the technology is and why
 to find"                                it exists"
"this breaks on Windows"               "where it sits in an architecture"
"our config, and why"                  "when not to use it"
"the error message and what it         "the trade-offs"
 actually meant"
```

> [!IMPORTANT]
> **Write the thing you would have wanted to read three hours ago.** Handbook notes are about reasoning and they age slowly; these notes are about specifics — a flag, an error message, a version incompatibility, the exact command that worked — and they save you the most time precisely because nobody else could have written them for you.

> [!TIP]
> **Write it the same day.** The detail that makes a note useful is the one you will have forgotten by tomorrow: the exact error string, the wrong assumption you started with, the search that finally worked. A note written a week later is a summary; a note written the same afternoon is a solution.

---

## Suggested entry shape

Keep it short. A useful note is often five lines.

```text
## <Technology> — <the specific thing>

DATE
CONTEXT      what I was trying to do
SYMPTOM      what happened, including the exact error text
CAUSE        what it actually was
FIX          the command, flag or config that worked
LESSON       the general form, if there is one
```

---

## Entries

*Append below. Newest first, or grouped by technology — whichever you will actually maintain.*

<!--
Example of the shape, to delete once you have real entries:

## Docker — build cache invalidated on every build

DATE      2026-04-11
CONTEXT   builds taking 3 minutes for a one-line source change
SYMPTOM   `npm ci` re-running on every build despite no dependency changes
CAUSE     `COPY . .` came BEFORE `RUN npm ci`, so any source change
          invalidated the dependency layer
FIX       copy package*.json first, install, then copy the rest
LESSON    order Dockerfile instructions from least to most frequently changed
          → see 13 - DevOps and Delivery/Docker.md section 5
-->

---

## Snippets I keep re-typing

*Commands, queries and configuration you look up more than twice. Put them here instead.*

```text
<language / tool>
<the snippet>
    → what it does, and when
```

---

## Version and environment notes

*Things that are true only for a specific version, on a specific platform. Date them — this section goes stale faster than anything else here.*

| Technology | Version | Note | Date |
|---|---|---|---|
| | | | |

> [!CAUTION]
> **Date every version-specific note, and delete the ones that are no longer true.** A note claiming a bug exists in a version you upgraded past a year ago will cost someone — probably you — an hour of confusion. This is the one section that is actively harmful when neglected.

---

## Things I have decided about my own stack

*Small, personal defaults. Not architecture decisions — those belong in an [ADR](../16%20-%20Templates/Architecture%20Decision%20Record.md).*

- **<default>** — <why>
- **<default>** — <why>

---

## Review prompt

*Read this file occasionally and ask:*

- [ ] Which entries are no longer true? Delete them.
- [ ] Which entry has come up three times? It belongs in a handbook note, not here.
- [ ] Which snippet should be a script, an alias, or a template in folder 16?
- [ ] Which lesson generalises? Add it to the relevant handbook note and link back.

# Questions

> **In one line —** the things you do not understand yet, written down instead of quietly avoided; this is the most valuable file in the handbook, because it is a map of your own edges.

| | |
|---|---|
| **Category** | Personal *(yours to fill — this file is scaffolding)* |
| **Related notes** | [Technology Notes](Technology%20Notes.md) · [Ideas](Ideas.md) · [Learning Philosophy](../00%20-%20Introduction/Learning%20Philosophy.md) · [How To Use This Handbook](../00%20-%20Introduction/How%20To%20Use%20This%20Handbook.md) |

---

## Why keep this file

```text
The normal fate of a question
    you half-understand something
    ↓
    you move on, because it mostly works
    ↓
    the gap becomes permanent, and invisible even to you
    ↓
    six months later you avoid a whole area of the system
    without ever having decided to
```

> [!IMPORTANT]
> **Writing a question down converts a vague discomfort into a specific thing you can answer.** "I don't really get databases" is unanswerable and demoralising; "why does adding an index on a foreign key sometimes make writes slower?" is a twenty-minute answer. The act of phrasing it precisely is most of the work — and it is why this file matters more than any note you will read.

> [!TIP]
> **Write the question the moment you notice it, even badly phrased.** A rough question captured is worth ten good questions forgotten. You can sharpen it later; you cannot recover it.

---

## The two kinds worth separating

```text
BLOCKING            you cannot finish today's work without this
                    → answer it now, then write down what you found

BACKGROUND          you would understand your systems better if you knew
                    → answer it deliberately, one per week
                    → this list is where actual expertise accumulates
```

---

## Blocking — answer now

*Questions in the way of current work. Delete each one as it is answered, having written the answer somewhere permanent.*

- [ ] <question>
- [ ] <question>

---

## Background — answer deliberately

*The interesting ones. Pick one a week. This list should never be empty; if it is, you have stopped noticing.*

- [ ] <question> — <why it matters to me>
- [ ] <question> — <why it matters to me>

<!--
Examples of the shape, to delete once you have real entries:

- [ ] Why does p99 latency rise so sharply above ~80% utilisation, when p50 barely moves?
      → I keep reading "queueing theory"; I want to actually understand the curve.
- [ ] What exactly happens between `git commit` and the object appearing in .git/objects?
      → I use Git daily and cannot describe the mechanism.
- [ ] Why is a Kubernetes Secret base64 rather than encrypted? What was the reasoning?
-->

---

## Answered

*Move questions here with the answer in one or two lines. This section is evidence, and it is worth re-reading.*

```text
## <question>
ANSWERED   YYYY-MM-DD
ANSWER     <one or two sentences, in your own words>
SOURCE     <where you found it>
LINK       <the handbook note you updated, if you did>
```

> [!TIP]
> **Answer in your own words, not by pasting a quotation.** If you cannot compress it into two sentences, you have found the answer without yet understanding it — which is a different question, and worth writing down as one.

---

## Things I thought I understood and did not

*A deliberately uncomfortable section, and the most useful one here.*

| I believed | Actually | Found out when |
|---|---|---|
| | | |

> [!IMPORTANT]
> **This table is where real learning is recorded.** A corrected misunderstanding is worth more than a new fact, because the wrong belief was actively shaping your decisions. It is also the section that makes you a better engineer to work with — someone who tracks what they got wrong argues from evidence rather than from confidence.

---

## Questions to ask someone else

*Things that are faster to ask than to research — a colleague, a maintainer, an issue tracker, a model. Batch them rather than interrupting constantly.*

- **<person or place>:** <question>

---

## Review prompt

*Read this file every couple of weeks and ask:*

- [ ] Which blocking questions did I work around instead of answering?
- [ ] Which background question has been here longest? Answer it or delete it honestly.
- [ ] Which answered question deserves a handbook note?
- [ ] Has anything moved into "thought I understood"? Write it down while it stings.
- [ ] Is this list empty? Then I have stopped noticing what I do not know.

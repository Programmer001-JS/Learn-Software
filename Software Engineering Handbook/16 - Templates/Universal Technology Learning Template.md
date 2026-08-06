# Universal Technology Learning Template

> **In one line —** the same 26 questions, asked of every technology; learn the questions once and every new tool becomes a form to fill in rather than a mountain to climb.

| | |
|---|---|
| **Category** | Template |
| **Use for** | Any technology, protocol, pattern or service you want to understand |
| **Related notes** | [How To Use This Handbook](../00%20-%20Introduction/How%20To%20Use%20This%20Handbook.md) · [Learning Philosophy](../00%20-%20Introduction/Learning%20Philosophy.md) · [Task Template](Task%20Template.md) · [Architecture Decision Record](Architecture%20Decision%20Record.md) |

---

## How to use this template

```text
1. Copy everything below the line marked "COPY FROM HERE"
2. Paste into a new note, named after the technology
3. Fill in what you can from documentation — 30-60 minutes
4. Leave 25 and 26 blank until you have actually USED the thing
5. Come back and finish them
```

> [!IMPORTANT]
> **Do not fill in all 26 sections for every technology.** Sections 1, 2, 3, 10, 16 and 19 are the load-bearing ones — what it is, why it exists, where it sits, when not to use it, the mental model, and the takeaway. If you only ever answer those six, you will still understand a technology better than most people who use it daily. The rest earn their place as you go deeper.

> [!TIP]
> **The single best test of whether a note is finished: close it and explain the technology out loud to an imaginary colleague.** Where you stumble is where the note is weak. Familiarity while reading feels exactly like knowledge and disappears the moment the file closes — writing sections 25 and 26 is what converts one into the other.

---

## The shape of a good answer

```text
TOO SHORT       "Redis is a cache."
                → true, and teaches nothing

TOO LONG        three paragraphs of documentation quoted verbatim
                → you have copied, not understood

RIGHT           "An in-memory key-value store. It exists because reading from
                 RAM is ~100× faster than from disk, and most applications read
                 the same small set of data repeatedly. It sits between the
                 application and the database. The cost is that it is not
                 durable by default, so it must never hold the only copy."
                → what, why, where, and the trade — in four sentences
```

---

## A note on the callouts

The handbook uses these deliberately, and copying the habit will make your own notes sharper:

```text
> [!IMPORTANT]   the one thing to remember if you forget everything else
> [!TIP]         the practical default; what an experienced person would do
> [!CAUTION]     the mistake that will actually bite you
> [!NOTE]        the analogy or mental model
```

> [!TIP]
> **Write at least one `[!CAUTION]` for every technology.** If you cannot name a way it goes wrong, you have read marketing material rather than learned a tool. The failure modes are what separate someone who has used a thing from someone who has read about it.

---

# ─────────── COPY FROM HERE ───────────

# <Technology Name>

> **In one line —** <one sentence that captures what it is AND its central trade-off. Write this LAST.>

| | |
|---|---|
| **Full name** | <if it is an abbreviation> |
| **Category** | <Message Queue / Relational Database / Container Runtime / …> |
| **Architectural Layer** | <Infrastructure / Data / Application / Integration / Security / Observability> |
| **Written in** | <optional, for tools where it matters> |
| **Sub-topics** | <for overview notes: links to the specific notes beneath it> |
| **Related notes** | <3-6 links; these connections are the real value of the handbook> |

---

## 1. Short Definition

*What is it?*

<Two to four sentences. Plain language. No marketing terms. If you cannot write this without using the tool's own vocabulary, you do not understand it yet.>

---

## 2. Problem

*What engineering problem does it solve?*

```text
<the situation without this technology>
    ↓
<what specifically goes wrong>
    ↓
<why the obvious alternative is inadequate>
```

> [!IMPORTANT]
> <The insight. Why does this thing exist rather than the simpler option someone would reach for first? Every technology was created because something specific was painful — name it.>

---

## 3. Architecture Position

*Where does it live in a system?*

```text
<a diagram showing what is above it, below it, and beside it>
```

<One or two sentences on what this position implies — what it can and cannot see, what depends on it.>

---

## 4. How It Works

*What is the mechanism?*

```text
<the core mechanism, in a diagram or a numbered sequence>
```

<Enough to reason about behaviour under load and under failure. Not an implementation walkthrough.>

---

## 5. Core Concepts

*What vocabulary do I need?*

```text
<TERM>        <what it means, in one line>
<TERM>        <what it means, in one line>
```

<The five to ten words you must know to read its documentation or discuss it with someone.>

---

## 6. Variants and Alternatives

*What else solves this, and how do they differ?*

| | Best for | The catch |
|---|---|---|
| **<this one>** | | |
| **<alternative>** | | |
| **<alternative>** | | |

> [!TIP]
> <Which one you would actually reach for by default, and why. A comparison with no recommendation is a shopping list, not a judgement.>

---

## 7. Configuration That Matters

*Which settings actually change behaviour?*

```text
<setting>       <what it does, and what a wrong value causes>
```

> [!CAUTION]
> <The default that is wrong for production, or the setting whose failure looks like something else entirely. Almost every technology has one.>

---

## 8. Real World Example

*Who uses it, and for what?*

- **<use case>** — <one line of context>
- **<use case>** — <one line of context>
- **<the case where it is the obvious choice>**

---

## 9. Communication and Dependencies

*What does it need, and what talks to it?*

- **<dependency>** — <why>
- **<dependency>** — <why>
- **<the dependency people forget until it breaks>**

---

## 10. When To Use / When NOT To Use

> [!TIP]
> <When this is the right answer. Be specific — "when you need X and can accept Y".>

> [!CAUTION]
> - **Not when <condition>** — <because>
> - **Not when <condition>** — <because>
> - **Not as a substitute for <the thing people misuse it as>**

---

## 11. Advantages and Disadvantages

**Advantages**
- <benefit>
- <benefit>

**Disadvantages**
- **<the one that matters most, in bold>**
- <cost>
- <operational burden>

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **<latency>** | <number or order of magnitude> |
| **<throughput>** | |
| **<the resource it is hungry for>** | |
| **<the hidden ceiling>** | |

> [!TIP]
> <The performance fact that is not obvious — the thing that throttles silently, or the metric to check first when it seems slow.>

---

## 13. Cost Impact

*What does it cost to run, and what drives that cost?*

```text
<cost driver>       <how it scales — per request, per GB, per hour>
```

> [!CAUTION]
> <The line item that surprises people. Egress, log ingestion, per-object requests, idle capacity — most technologies have one.>

---

## 14. Security Considerations

> [!CAUTION]
> <The one security failure that actually happens with this technology in the real world. Not a generic reminder — the specific, documented, recurring mistake.>

- **<control>** — <why>
- **<control>** — <why>
- **<the default that is insecure>**

---

## 15. Failure Modes

*How does it break, and what does it look like?*

```text
<failure>       <symptom you will observe>  →  <what it actually is>
<failure>       <symptom>                   →  <cause>
```

> [!IMPORTANT]
> <How it degrades: gracefully, or all at once? Systems that fail abruptly need headroom and limits; systems that degrade need alarms on the degradation.>

---

## 16. Mental Model

> [!NOTE]
> **<A one-sentence analogy from the physical world.>**
>
> <Two to four sentences developing it — including where the analogy breaks down, which is usually the most instructive part.>

---

## 17. Mini Architecture Diagram

```text
<a fuller diagram than section 3: the realistic production shape,
 with the surrounding components you would actually deploy>
```

---

## 18. Complete Request Flow

```text
<follow ONE request or operation end to end>
    ↓
<step>
    ↓
─────────────── under load ───────────────
<what changes>
    ↓
─────────────── when something fails ───────────────
<what happens, and what recovers it>
```

> [!TIP]
> **This section is the most valuable one to write and the one most often skipped.** Tracing a single operation from arrival to response, then again under failure, exposes every gap in your understanding — you cannot write it vaguely.

---

## 19. Key Takeaway

> [!IMPORTANT]
> <One sentence. If you remembered nothing else about this technology, this is what you would want to know.>

---

## 20. Common Mistakes

- **<mistake>** — <consequence>
- **<mistake>** — <consequence>
- **<the mistake you personally made>**

---

## 21. Open Source Technologies

- **<tool>** — <what it does for you>
- **<tool>** — <what it does for you>
- **<the tool that makes daily operation bearable>**

---

## 22. Learning Resources

- **<the official documentation page that is actually good>**
- **<the paper, talk or post that explains the WHY>**
- **<the one to skip, and why>**

---

## 23. Explanation Test

*Can I explain this without notes?*

- [ ] What is it, in two sentences?
- [ ] What problem does it solve, and what did people do before?
- [ ] Where does it sit in an architecture?
- [ ] Name one alternative and say why you would not choose it.
- [ ] Name one situation where using this would be a mistake.
- [ ] Describe how it fails.

---

## 24. Related Notes

- [<note>](<path>) — <how it connects>
- [<note>](<path>) — <how it connects>

> [!TIP]
> **Add links whenever you notice a connection, even if the target note does not exist yet.** The handbook's value grows with the number of connections, not the number of files — and a link to a note you have not written is a useful reminder that you should.

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**
- **Where this surprised me:**

---

## 26. Workbook Exercise

*One practical task. Leave it unchecked until it is done.*

- [ ] <something you actually run, break or measure>
- [ ] <something you check in your own system>
- [ ] <a number you write down>

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
<the whole thing in one or two lines>
```

## 2. Request Flow

```text
Input       <what goes in>
    ↓
Processing  <what happens to it>
    ↓
Output      <what comes out, and what guarantee it carries>
```

## 3. Real-World Usage

<Two to four sentences on how this is actually used in production, and what the industry learned about using it well. This is where you say what the documentation will not.>

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | |
| **Why does it exist?** | |
| **Where does it belong?** | |
| **When should I use it?** | |

# ─────────── COPY TO HERE ───────────

---

## The short version, for when you have 20 minutes

If a full note is too much for the technology in front of you, answer only these six:

```text
1. What is it?                      (two sentences)
2. Why does it exist?               (what was painful before)
3. Where does it sit?               (one diagram)
4. When should I NOT use it?        (the most useful question here)
5. What is the mental model?        (one analogy)
6. What is the one takeaway?        (one sentence)
```

> [!IMPORTANT]
> **Question 4 is the one that distinguishes understanding from familiarity.** Anyone can list what a tool does; knowing where it is the wrong choice requires knowing its costs, and that only comes from having thought about the trade-off rather than the feature list.

---

## Personal Notes

*Your notes on the template itself — what you added, what you never use.*

- **Sections I always fill in:**
- **Sections I never use:**
- **Sections I added of my own:**

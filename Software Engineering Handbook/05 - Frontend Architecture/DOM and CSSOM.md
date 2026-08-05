# DOM and CSSOM

> **In one line —** the browser's live, in-memory representation of your page structure (DOM) and its styles (CSSOM); JavaScript changes these, and the screen follows.

| | |
|---|---|
| **Full names** | Document Object Model · CSS Object Model |
| **Category** | Browser Data Structure |
| **Architectural Layer** | Client |
| **Related notes** | [Rendering Engine](Rendering%20Engine.md) · [Browser Internals](Browser%20Internals.md) · [HTML](HTML.md) · [CSS](CSS.md) · [React](React.md) |

---

## 1. Short Definition

*What is it?*

The **DOM** is a tree of objects representing the page's structure, built from the HTML. The **CSSOM** is the parallel tree of style rules built from the CSS. Together they produce what is drawn.

---

## 2. Purpose

*What is its main purpose?*

To turn static text documents into **live, programmable objects**. Once HTML is parsed, it is no longer text — it is a tree that JavaScript can read and modify, and the browser re-renders in response.

---

## 3. Problem

*What engineering problem does it solve?*

A page needs to change after it loads — a menu opens, data arrives, a form validates. Without an in-memory model, changing anything would mean re-downloading and re-parsing the whole document.

```text
HTML text  →  parse once  →  DOM tree  →  mutate freely, forever
```

---

## 4. Architecture Position

```text
HTML                        CSS
 ↓ parse                     ↓ parse
DOM                        CSSOM
 └───────────┬───────────────┘
             ↓
       Render tree  → Layout → Paint → Composite
             ↑
      JavaScript mutations
```

---

## 5. Structure

```html
<html>
  <body>
    <div class="card">
      <h1>Title</h1>
    </div>
  </body>
</html>
```

```text
Document
   └── html
        └── body
             └── div.card
                  └── h1
                       └── "Title"    ← text nodes are nodes too
```

Every element, attribute and piece of text is an object with properties and methods.

---

## 6. Why DOM manipulation is slow

> [!IMPORTANT]
> The DOM is not slow to *read*. What is expensive is that changing it can trigger **layout and paint** in the [rendering engine](Rendering%20Engine.md) — and doing that in a loop, hundreds of times, is what actually costs you.

```text
SLOW                                    FAST
for (item of items) {                   const frag = document.createDocumentFragment();
  list.appendChild(makeEl(item));       for (item of items) frag.appendChild(makeEl(item));
}                                       list.appendChild(frag);
↑ potential layout per iteration        ↑ one insertion, one layout
```

---

## 7. Layout thrashing again

```javascript
// Every read after a write forces a synchronous layout
el.style.width = '100px';
const h = el.offsetHeight;   // ← forces layout NOW
el.style.height = h + 'px';
const w = el.offsetWidth;    // ← forces layout AGAIN
```

> [!TIP]
> **Batch reads, then batch writes.** Properties like `offsetHeight`, `getBoundingClientRect()` and `scrollTop` all force the browser to flush pending layout work immediately.

---

## 8. The virtual DOM

Frameworks such as [React](React.md) keep their own lightweight copy of the tree.

```text
State changes
    ↓
Build a new virtual tree  (plain JavaScript objects — cheap)
    ↓
DIFF against the previous virtual tree
    ↓
Apply only the minimal set of real DOM changes
```

> [!TIP]
> The virtual DOM is not faster than hand-written optimal DOM code — it is faster than the code most people would write, and far easier to reason about. That is the actual trade.

---

## 9. Real World Example

- **Every JavaScript framework** exists to manage DOM updates: React diffs a virtual tree, Svelte compiles direct DOM operations, Vue tracks reactive dependencies.
- **`document.querySelector`** is the DOM's public API — the same tree the browser renders from.
- **Server-side rendering** sends pre-built HTML so the DOM exists before JavaScript loads.

---

## 10. Input, Processing, Output

**Input:** HTML and CSS text, plus JavaScript mutations.
**Processing:** parsed into trees; style rules matched to elements; changes marked dirty.
**Output:** a render tree that layout and paint consume.

---

## 11. Communication and Dependencies

- **[HTML](HTML.md) parser** builds the DOM incrementally as bytes arrive
- **[CSS](CSS.md) parser** builds the CSSOM — and **CSS blocks rendering**, because no element can be drawn before its style is known
- **[JavaScript engine](JavaScript%20Engine.md)** mutates both
- **[Rendering engine](Rendering%20Engine.md)** consumes the combined result

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Direct DOM manipulation is fine for small, targeted changes and for vanilla-JS pages. Use a framework when UI state grows complex enough that manual synchronisation becomes error-prone.

> [!CAUTION]
> Do not mix manual DOM manipulation with a framework that owns the same elements. React and jQuery editing the same nodes produces bugs that are extremely hard to reproduce, because two systems disagree about what the tree contains.

---

## 13. Advantages and Disadvantages

**Advantages**
- Live and programmable — everything on screen is reachable
- Standardised across browsers
- Event system with bubbling and delegation built in

**Disadvantages**
- Mutations can be expensive because of layout and paint
- Very large trees slow down every operation
- The API is verbose compared with modern alternatives
- Easy to leak memory by holding references to detached nodes

---

## 14. Performance Impact

| Operation | Cost |
|---|---|
| Reading a property | Cheap |
| Reading geometry (`offsetHeight`) | **Forces layout** |
| Adding or removing an element | Layout + paint |
| Changing `transform` / `opacity` | Composite only — cheap |
| A tree with 10,000+ nodes | Everything gets slower |

---

## 15. Security Considerations

> [!CAUTION]
> **`innerHTML` with user-supplied content is the classic XSS vulnerability.** It parses the string as HTML, so `<img src=x onerror="...">` executes. Use `textContent` for text, and sanitise deliberately when HTML really is required.

- **`textContent` is safe; `innerHTML` is not**
- **Content Security Policy** blocks inline scripts and is the strongest structural defence against XSS
- **`dangerouslySetInnerHTML`** in React is named that way on purpose
- Anything readable in the DOM is readable by any script on the page — including injected ones

---

## 16. Mental Model

> [!NOTE]
> **The DOM is a live blueprint of a building that is already standing.**
>
> Change the blueprint and the building rearranges itself immediately. Small changes are quick; moving a wall means everything around it has to shift, which is why some edits cost far more than others.

---

## 17. Mini Architecture Diagram

```text
HTML bytes                 CSS bytes
    ↓                          ↓
  DOM tree                 CSSOM tree
    └────────────┬─────────────┘
                 ↓
           Render tree
                 ↓
        Layout → Paint → Composite
                 ↑
          JavaScript (mutations, events)
```

---

## 18. Complete Request Flow

```text
HTML arrives and is parsed incrementally → DOM grows
    ↓
<link rel="stylesheet"> found → CSS fetched → RENDERING BLOCKED until it arrives
    ↓
CSSOM built
    ↓
DOM + CSSOM → render tree (display:none elements excluded)
    ↓
Layout → Paint → first pixels on screen
    ↓
JavaScript loads and mutates the DOM
    ↓
Affected subtree marked dirty → layout/paint/composite as needed
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> The DOM is a live object tree, and its cost is not in reading it but in the layout and paint work that mutating it triggers — so batch changes and prefer compositor-only properties.

---

## 20. Common Mistakes

- **Using `innerHTML` with user input** — XSS
- **Mutating the DOM inside a loop** instead of batching
- **Interleaving geometry reads and style writes** — layout thrashing
- **Mixing framework-managed and manual DOM updates**
- **Holding references to removed nodes**, leaking memory
- **Enormous DOM trees** — every operation pays for them

---

## 21. Open Source Technologies

- **React**, **Vue**, **Svelte** — abstractions over DOM updates
- **DOMPurify** — sanitise HTML before inserting it
- **jsdom** — a DOM implementation for testing outside a browser
- **Chrome DevTools Elements panel** — inspect the live tree

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Insert 1,000 elements one by one, then with a DocumentFragment, and compare the timings.
- [ ] Search your codebase for `innerHTML` and check whether any of it receives user input.
- [ ] Count the DOM nodes on your heaviest page (`document.querySelectorAll('*').length`).

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
HTML → DOM
CSS  → CSSOM
   ↓
Render tree → Layout → Paint → Screen
```

## 2. Request Flow

```text
Input       HTML and CSS text, plus JavaScript mutations
    ↓
Processing  parsed into object trees, combined into a render tree
    ↓
Output      a structure the rendering pipeline draws from
```

## 3. Real-World Usage

**React's virtual DOM** exists specifically because direct DOM mutation is easy to do badly. Diffing plain JavaScript objects and applying a minimal set of real changes trades a little CPU for a great deal of correctness.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Live object trees representing page structure and styles |
| **Why does it exist?** | So a loaded page can be changed without being reloaded |
| **Where does it belong?** | Between parsing and rendering, inside the browser |
| **When should I use it?** | Constantly — the skill is minimising expensive mutations |

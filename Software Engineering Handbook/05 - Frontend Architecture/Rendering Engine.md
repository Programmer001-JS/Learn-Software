# Rendering Engine

> **In one line —** the part of the browser that turns HTML and CSS into pixels, and the reason some CSS properties are cheap to animate and others destroy your frame rate.

| | |
|---|---|
| **Category** | Browser Component |
| **Architectural Layer** | Client |
| **Examples** | Blink (Chrome, Edge) · WebKit (Safari) · Gecko (Firefox) |
| **Related notes** | [Browser Internals](Browser%20Internals.md) · [DOM and CSSOM](DOM%20and%20CSSOM.md) · [CSS](CSS.md) · [Event Loop](Event%20Loop.md) |

---

## 1. Short Definition

*What is it?*

The rendering engine parses HTML and CSS, works out where every element goes and what it looks like, and draws it. It is the largest and most complex part of a browser.

---

## 2. Purpose

*What is its main purpose?*

To convert a declarative description of a document into an exact arrangement of coloured pixels — repeatedly, up to sixty times per second, while the page changes underneath it.

---

## 3. The pipeline

```text
HTML                          CSS
 ↓ parse                       ↓ parse
DOM                          CSSOM
 └──────────┬─────────────────┘
            ↓
      RENDER TREE      only visible elements, with computed styles
            ↓
        LAYOUT         geometry: position and size of every box
            ↓
        PAINT          fill in pixels: colours, text, shadows, borders
            ↓
      COMPOSITE        assemble layers, usually on the GPU
            ↓
         SCREEN
```

> [!IMPORTANT]
> **Each stage is more expensive than the one after it.** Changing something that forces layout re-runs paint and composite too. Changing something that only affects compositing skips both. This single fact explains most frontend performance advice.

---

## 4. Reflow and repaint

```text
REFLOW  (layout)     geometry changed — width, height, position, font-size
                     → recalculates layout for affected elements, then repaints
                     → EXPENSIVE

REPAINT              appearance changed — background-color, visibility
                     → skips layout
                     → moderate

COMPOSITE ONLY       transform, opacity
                     → GPU moves an existing layer
                     → CHEAP
```

> [!TIP]
> This is why the standard advice is to animate with `transform` and `opacity` rather than `top`, `left`, `width` or `height`. It is not stylistic preference — the first pair skips two entire pipeline stages.

---

## 5. Layout thrashing

```text
// forces a reflow on EVERY iteration
for (const el of elements) {
  el.style.width = el.offsetWidth + 10 + 'px';
  //               ↑ reading offsetWidth forces layout to be recalculated NOW
}
```

Reading a geometric property after writing one forces the browser to flush layout synchronously. Doing that in a loop can turn a 1 ms operation into 500 ms.

> [!TIP]
> **Read all measurements first, then write all changes.** Batching in that order lets the browser do one layout pass instead of hundreds.

---

## 6. Real World Example

- **Smooth scrolling and animation** in any well-built site is the result of staying on the compositor-only path.
- **The Core Web Vitals metrics** map directly onto this pipeline: **LCP** measures when the main content finished painting, **CLS** measures unexpected layout shifts, **INP** measures how quickly interaction produces a visual response.
- **Reserving space for images** (`width` and `height` attributes) prevents layout shift — a pure rendering-engine concern with a direct SEO consequence.

---

## 7. Input, Processing, Output

**Input:** HTML bytes, CSS rules, fonts, images, and DOM mutations from JavaScript.

**Processing:** parse → build trees → compute styles → layout → paint → composite.

**Output:** a rendered frame, ideally every 16.7 ms.

---

## 8. Internal Idea

*How does it work internally?*

Rendering is organised into **layers**. Certain properties promote an element to its own compositor layer, which the GPU can then move, scale or fade without the CPU touching layout or paint at all.

```text
Normal element      changing it → layout → paint → composite
Own layer           changing transform/opacity → composite only
```

> [!CAUTION]
> Promoting too many elements to layers (`will-change: transform` everywhere) consumes GPU memory and can make performance worse. It is a targeted tool, not a global setting.

---

## 9. Communication and Dependencies

- **[DOM and CSSOM](DOM%20and%20CSSOM.md)** — its input trees
- **[JavaScript engine](JavaScript%20Engine.md)** — mutates the DOM and triggers re-rendering
- **[Event loop](Event%20Loop.md)** — rendering happens between tasks, so a blocked loop blocks rendering
- **GPU process** — final compositing

---

## 10. When To Use / When NOT To Use

> [!TIP]
> You do not choose a rendering engine — you write CSS and JavaScript that either cooperate with it or fight it. The practical skill is knowing which changes are cheap.

> [!CAUTION]
> Avoid: animating layout properties, reading geometry inside loops, deeply nested CSS selectors on large DOM trees, and DOM trees with tens of thousands of nodes. Each of these makes the engine do dramatically more work than necessary.

---

## 11. Advantages and Disadvantages

**Advantages**
- Declarative — you describe the result, not the drawing steps
- Heavily optimised over decades
- GPU compositing makes smooth animation possible on modest hardware

**Disadvantages**
- The cost model is invisible unless you know it
- Small CSS or JS changes can have order-of-magnitude performance effects
- Engine differences still cause real cross-browser bugs

---

## 12. Performance Impact

| Stage | Relative cost | Triggered by |
|---|---|---|
| **Style recalculation** | Low–medium | Class changes, new CSS |
| **Layout** | **High** | Geometry changes, DOM insertion, font loading |
| **Paint** | Medium | Colours, shadows, borders |
| **Composite** | Low | `transform`, `opacity` |

---

## 13. Security Considerations

> [!CAUTION]
> The rendering engine parses hostile input from the entire internet, in C++, at extremely high speed. It is historically one of the largest sources of remote-code-execution vulnerabilities in consumer software — which is why browsers auto-update aggressively and sandbox renderer processes.

- Rendering engine bugs have repeatedly been used in real-world drive-by attacks
- This is the direct justification for [site isolation](Browser%20Internals.md) and sandboxing
- As an application developer: keep users on current browsers, and set a **Content Security Policy**

---

## 14. Mental Model

> [!NOTE]
> **The rendering engine is a stage crew setting up a theatre.**
>
> **Layout** is deciding where every piece of scenery stands — move one and everything around it must be repositioned. **Paint** is decorating each piece. **Compositing** is sliding a finished piece across the stage on wheels: fast, because nothing has to be rebuilt.

---

## 15. Mini Architecture Diagram

```text
HTML → DOM ─┐
            ├→ Render Tree → Layout → Paint → Composite → Screen
CSS → CSSOM ┘                  ↑                  ↑
                          expensive            GPU, cheap
                               ↑
                    JavaScript mutations
```

---

## 16. Complete Request Flow

A button click that expands a panel:

```text
Click event → event loop → JavaScript handler
    ↓
JS adds a CSS class
    ↓
Style recalculation for affected elements
    ↓
Does the change affect geometry?
    ↓ YES → LAYOUT (expensive) → PAINT → COMPOSITE
    ↓ NO, only transform/opacity → COMPOSITE ONLY (cheap)
    ↓
Frame rendered — hopefully within the 16.7 ms budget
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> Rendering is a pipeline where each stage is more expensive than the next — animate `transform` and `opacity` to skip layout and paint entirely.

---

## 18. Common Mistakes

- **Animating `left`, `top`, `width` or `height`** instead of `transform`
- **Layout thrashing** — interleaving reads and writes of geometry
- **Not reserving space for images**, causing layout shift
- **Applying `will-change` broadly**, exhausting GPU memory
- **Enormous DOM trees** — every extra node costs on every layout pass
- **Assuming a fast laptop represents your users**

---

## 19. Open Source Technologies

- **Blink**, **WebKit**, **Gecko** — the three engines that matter
- **Chrome DevTools Performance panel** — see the pipeline stages directly
- **Lighthouse**, **WebPageTest** — measure Core Web Vitals
- **web-vitals** — collect real-user metrics in production

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Animate an element with `left` and then with `transform`, and compare frame rates in DevTools.
- [ ] Record a performance profile of your own site and find the longest layout operation.
- [ ] Find one image on your site without explicit dimensions and measure the layout shift it causes.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
HTML + CSS
    ↓
DOM + CSSOM
    ↓
Render tree → Layout → Paint → Composite
    ↓
Screen
```

## 2. Request Flow

```text
Input       HTML, CSS and DOM mutations
    ↓
Processing  style → layout → paint → composite
    ↓
Output      a rendered frame within 16.7 ms
```

## 3. Real-World Usage

**Google's Core Web Vitals** turned rendering-engine behaviour into an SEO ranking factor. Cumulative Layout Shift is literally a measure of how often the layout stage moved content unexpectedly — an internal browser concept that now affects business outcomes.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The browser component that turns HTML and CSS into pixels |
| **Why does it exist?** | To render a declarative document description efficiently and repeatedly |
| **Where does it belong?** | Inside the browser's renderer process |
| **When should I use it?** | Understand its cost model whenever frontend smoothness matters |

# CSS

> **In one line —** the language that decides how everything looks and where it goes, using a cascade and a specificity system that is the source of most frontend frustration.

| | |
|---|---|
| **Full name** | Cascading Style Sheets |
| **Category** | Styling Language |
| **Architectural Layer** | Client / Presentation |
| **Related notes** | [HTML](HTML.md) · [DOM and CSSOM](DOM%20and%20CSSOM.md) · [Rendering Engine](Rendering%20Engine.md) · [Frontend Architecture](Frontend%20Architecture.md) |

---

## 1. Short Definition

*What is it?*

CSS is a declarative language that describes how HTML elements should be presented: colours, spacing, typography, layout and animation. You state the desired result; the [rendering engine](Rendering%20Engine.md) works out how to draw it.

---

## 2. Purpose

*What is its main purpose?*

To separate **presentation from content**, so the same HTML can be restyled entirely, adapted to different screen sizes, and maintained without touching the markup.

---

## 3. Problem

*What engineering problem does it solve?*

Before CSS, appearance was embedded in the markup — `<font>` tags and layout tables everywhere. Changing a colour meant editing every page, and the same document could not adapt to different devices.

---

## 4. Architecture Position

```text
HTML  (structure)
   ↓
CSS   (presentation)   ← you are here
   ↓
CSSOM
   ↓
Render tree → Layout → Paint → Composite
```

> [!IMPORTANT]
> **CSS blocks rendering.** The browser will not paint anything until it has the stylesheets it needs, because drawing unstyled content and restyling it would produce a visible flash. This is why stylesheet size and placement matter so much.

---

## 5. The cascade and specificity

When several rules target the same element, CSS resolves the conflict in a defined order:

```text
1. Importance     !important  wins over everything
2. Specificity    inline (1000) > id (100) > class (10) > element (1)
3. Source order   later wins among equals
```

```text
#nav .item a       →  100 + 10 + 1  = 111
.nav .item a       →   10 + 10 + 1  =  21
a                  →                =   1
```

> [!CAUTION]
> **Specificity wars are the classic CSS failure mode.** One `!important` leads to another, and eventually nothing can be overridden without escalating further. Keep specificity low and flat — that is what utility-first and BEM approaches are really solving.

---

## 6. Layout — the two systems that matter

```text
FLEXBOX     one dimension: a row or a column
            navigation bars, toolbars, centring, distributing space

GRID        two dimensions: rows and columns together
            page layouts, card grids, complex arrangements
```

Everything before these — floats, absolute positioning for layout, table hacks — is now legacy. Flexbox and Grid solved layout properly.

---

## 7. The box model

```text
┌─────────── margin ────────────┐
│  ┌───────── border ────────┐  │
│  │  ┌────── padding ─────┐ │  │
│  │  │      content       │ │  │
│  │  └────────────────────┘ │  │
│  └─────────────────────────┘  │
└───────────────────────────────┘
```

> [!TIP]
> `box-sizing: border-box` makes `width` include padding and border, which is what almost everyone intends. Setting it globally is standard practice in essentially every modern project.

---

## 8. Responsive design

```css
/* Mobile first: base styles, then add for larger screens */
.card { width: 100%; }

@media (min-width: 768px) {
  .card { width: 50%; }
}
```

> [!TIP]
> Write mobile-first. Most traffic is mobile, and adding complexity for larger screens is easier than stripping it away for smaller ones.

---

## 9. Real World Example

- **Design systems** (Material, Tailwind, Bootstrap) are CSS conventions packaged so that large teams stay visually consistent.
- **Dark mode** is `prefers-color-scheme` plus CSS custom properties — a few lines rather than a second stylesheet.
- **Animating `transform` instead of `left`** is the single most repeated piece of CSS performance advice, and it comes directly from the [rendering pipeline](Rendering%20Engine.md).

---

## 10. Approaches to organising CSS

| Approach | Idea | Trade-off |
|---|---|---|
| **Plain CSS + BEM** | Naming convention for flat specificity | Verbose, but predictable |
| **CSS Modules** | Locally scoped class names | Build step required |
| **Tailwind** | Utility classes in the markup | Fast, but markup gets noisy |
| **CSS-in-JS** | Styles co-located with components | Runtime cost in some libraries |
| **Custom properties** | Native CSS variables | Now the standard for theming |

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use CSS for everything presentational, including animation and simple state (`:hover`, `:focus`, `:checked`). CSS-driven behaviour is faster and more reliable than JavaScript equivalents.

> [!CAUTION]
> Do not use JavaScript to do what CSS already does — animating with `setInterval`, calculating layout manually, or toggling inline styles where a class would do. And do not treat CSS as a security boundary: `display: none` hides content visually, it does not remove it from the page.

---

## 12. Advantages and Disadvantages

**Advantages**
- Declarative and highly expressive
- Complete separation of presentation from content
- Fast — the browser optimises it heavily
- Responsive and adaptive with no JavaScript

**Disadvantages**
- Global by default; scoping needs discipline or tooling
- Specificity and the cascade are genuinely hard to reason about at scale
- Easy to accumulate dead rules nobody dares to remove
- Debugging "why is this element that colour" can take real time

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Render blocking** | Rendering waits for CSS — keep critical CSS small |
| **File size** | Large frameworks ship far more than most pages use |
| **Selector complexity** | Deeply nested selectors cost on large DOM trees |
| **Animated properties** | `transform`/`opacity` are cheap; `width`/`top` force layout |

> [!IMPORTANT]
> Animate **`transform`** and **`opacity`**. Everything else risks running layout and paint on every frame, and 16.7 ms disappears quickly.

---

## 14. Security Considerations

> [!CAUTION]
> CSS is not a security mechanism. Content hidden with `display: none` is still fully present in the [DOM](DOM%20and%20CSSOM.md) and visible to anyone who opens DevTools. Never "hide" data the user is not allowed to see — do not send it.

- **CSS injection** from user-controlled styles can exfiltrate data through attribute selectors and background URLs
- **`@import` from untrusted sources** loads third-party code into your page context
- **Content Security Policy** should restrict `style-src` as well as `script-src`
- **Clickjacking** is a CSS-and-iframe technique; defend with `X-Frame-Options` or CSP `frame-ancestors`

---

## 15. Mental Model

> [!NOTE]
> **CSS is a set of instructions to a decorator, plus a rulebook for what happens when two instructions conflict.**
>
> The cascade is that rulebook. `!important` is shouting — it works once, and then everyone starts shouting and nobody can be heard.

---

## 16. Mini Architecture Diagram

```text
CSS files
    ↓ parse
CSSOM
    ↓ matched against
DOM
    ↓
Render tree → Layout → Paint → Composite
```

---

## 17. Complete Request Flow

```text
HTML parsed, <link rel="stylesheet"> discovered
    ↓
CSS downloaded — RENDERING IS BLOCKED
    ↓
CSSOM built
    ↓
Selectors matched against every element
    ↓
Computed styles resolved via the cascade and specificity
    ↓
Render tree built → layout → paint
    ↓
Class changed later by JavaScript
    ↓
Only the affected subtree is recalculated
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> CSS separates presentation from content, and its cascade means conflicts are resolved by specificity rather than intent — keep specificity low, and animate only compositor-friendly properties.

---

## 19. Common Mistakes

- **`!important` as a first resort**
- **Deeply nested selectors** that are impossible to override sensibly
- **Animating layout properties** instead of `transform`/`opacity`
- **Shipping an entire framework** for a handful of components
- **Desktop-first media queries**, then fighting them on mobile
- **Using `display: none` to "protect" data**
- **Never deleting dead CSS**, because nobody is sure what uses it

---

## 20. Open Source Technologies

- **Tailwind CSS**, **Bootstrap**, **Bulma** — utility and component frameworks
- **PostCSS**, **Sass** — preprocessing and tooling
- **CSS Modules**, **styled-components**, **vanilla-extract** — scoping strategies
- **PurgeCSS** — remove unused rules
- **stylelint** — enforce consistency

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Find every `!important` in your project and work out what each one is fighting.
- [ ] Rebuild one layout using Grid instead of whatever it currently uses.
- [ ] Animate an element with `left`, then with `transform`, and compare frame rates in DevTools.

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
Input       CSS rules and a DOM tree
    ↓
Processing  selectors matched, cascade and specificity resolved, styles computed
    ↓
Output      a styled render tree ready for layout and paint
```

## 3. Real-World Usage

**Every design system** in production — Material, Tailwind, or an in-house one — exists to impose consistency on CSS's global nature. The technical problem being solved is always the same: the cascade does not scale without conventions.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A declarative language for presentation and layout |
| **Why does it exist?** | To separate how content looks from what content is |
| **Where does it belong?** | Between HTML and the rendering pipeline |
| **When should I use it?** | For all presentation — and never as a security boundary |

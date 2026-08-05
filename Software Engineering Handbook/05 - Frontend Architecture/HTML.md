# HTML

> **In one line —** the language that describes what content *means*, not how it looks — and getting that meaning right is what makes a page accessible, findable and maintainable.

| | |
|---|---|
| **Full name** | HyperText Markup Language |
| **Category** | Markup Language |
| **Architectural Layer** | Client / Content |
| **Related notes** | [CSS](CSS.md) · [DOM and CSSOM](DOM%20and%20CSSOM.md) · [Browser Internals](Browser%20Internals.md) · [React](React.md) |

---

## 1. Short Definition

*What is it?*

HTML is a markup language that describes the **structure and meaning** of content using tags: this is a heading, this is a navigation region, this is a button, this is a form field.

---

## 2. Purpose

*What is its main purpose?*

To give content semantic structure so that browsers, screen readers, search engines and other tools can all understand it — each in their own way — from the same source.

---

## 3. Problem

*What engineering problem does it solve?*

A screen reader, a search crawler and a browser need to interpret the same document differently. Without semantic markup, everything is an undifferentiated `<div>` and none of them can tell a heading from a caption.

```text
<div class="header-big">Title</div>     ← visually a heading, semantically nothing
<h1>Title</h1>                          ← a heading to every tool that reads it
```

---

## 4. Architecture Position

```text
Server  →  HTML  →  parsed into the DOM
                        ↓
              CSS styles it   ·   JavaScript makes it interactive
                        ↓
                     Rendered page
```

> [!IMPORTANT]
> HTML is the **content layer**. CSS is presentation, JavaScript is behaviour. Keeping the three separated is the oldest and still one of the most useful frontend principles.

---

## 5. Semantic elements

```html
<header>    site or section header
<nav>       navigation links
<main>      the primary content — one per page
<article>   self-contained content
<section>   a thematic grouping
<aside>     tangentially related content
<footer>    footer for a section or the page
<button>    something that performs an action
<a>         something that navigates somewhere
```

> [!TIP]
> **`<button>` versus `<div onclick>`** is the clearest example of why this matters. A real button is focusable, works with the keyboard, is announced correctly by screen readers, and submits forms. A `div` with a click handler does none of that until you reimplement all of it — usually incompletely.

---

## 6. Accessibility

Roughly 15% of people have some form of disability. Semantic HTML is most of what accessibility requires, and it is free if you write it correctly from the start.

- **Every image needs `alt`** — descriptive, or empty (`alt=""`) if purely decorative
- **Every input needs a `<label>`**
- **Headings must nest logically** — `h1` → `h2` → `h3`, never skipping for visual size
- **Colour must not be the only signal** for meaning
- **Everything interactive must work with the keyboard**

> [!CAUTION]
> Accessibility is also a **legal requirement** in many jurisdictions, and retrofitting it into a finished application costs far more than building it in.

---

## 7. Forms

Forms are where HTML does the most work for you:

```html
<label for="email">Email</label>
<input type="email" id="email" name="email" required>
```

`type="email"` gives you a suitable mobile keyboard, browser validation, and correct semantics — three things for one attribute.

> [!CAUTION]
> **Browser validation is a convenience, never a security control.** Anyone can send a request without ever loading your page. Validate on the server, always. See [Validation](../07%20-%20Backend%20Design%20Patterns/Validation.md).

---

## 8. Real World Example

- **SEO** depends heavily on semantic structure: headings, `<title>`, meta descriptions, and structured data.
- **Screen readers** navigate by landmarks (`<nav>`, `<main>`) and headings — a page of `<div>`s is effectively unnavigable.
- **Reader modes** in browsers rely on `<article>` to identify the real content.
- **Social previews** come from Open Graph `<meta>` tags.

---

## 9. Input, Processing, Output

**Input:** an HTML text document.
**Processing:** parsed incrementally into the [DOM](DOM%20and%20CSSOM.md); the parser is extremely tolerant and recovers from malformed markup rather than failing.
**Output:** a DOM tree that CSS styles and JavaScript manipulates.

---

## 10. Communication and Dependencies

- **[CSS](CSS.md)** — targets elements by tag, class and attribute
- **[JavaScript](JavaScript.md)** — reads and mutates the resulting DOM
- **Servers** — deliver it, whether static, server-rendered or generated
- **Assistive technology and crawlers** — consume its semantics

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Always start from semantic HTML, then style it. Choosing the element by meaning first — and appearance second — is a habit that pays continuously.

> [!CAUTION]
> Do not use HTML for layout (tables for page structure, `<br>` for spacing) or for styling (`<b>`, `<center>`, inline `style` everywhere). That is what CSS is for, and mixing them makes both harder to change.

---

## 12. Advantages and Disadvantages

**Advantages**
- Extremely forgiving parser — a malformed page still renders
- Accessibility and SEO largely come for free with correct semantics
- Universal, stable and backwards-compatible for decades
- Declarative and readable

**Disadvantages**
- The permissive parser hides mistakes rather than surfacing them
- Verbose
- Easy to produce technically valid but semantically meaningless markup
- No built-in templating or componentisation

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Document size** | Larger HTML means more to download and parse |
| **DOM node count** | Every node costs on every layout pass |
| **Blocking resources** | `<script>` without `defer` blocks parsing; CSS blocks rendering |
| **Images** | Setting `width`/`height` prevents layout shift |

> [!TIP]
> Put `<link rel="stylesheet">` early, and use `defer` on scripts. Those two habits solve most render-blocking problems.

---

## 14. Security Considerations

> [!CAUTION]
> **HTML injection is XSS.** Inserting user-supplied text into a page without escaping lets an attacker inject `<script>` and run code as your user.

- **Escape all user content** on output; every serious template engine does this by default — do not disable it
- **Sanitise** with a library such as DOMPurify when you genuinely must allow HTML
- **`rel="noopener noreferrer"`** on external links with `target="_blank"`, or the opened page can manipulate yours
- **Content Security Policy** is the structural defence that limits damage when escaping fails
- **`<iframe>` needs `sandbox`** when embedding anything you do not control

---

## 15. Mental Model

> [!NOTE]
> **HTML is the skeleton, CSS is the skin, JavaScript is the muscles.**
>
> The skeleton determines what the body *is* — where the joints are and what can move. You can change the skin freely. Building a body without a proper skeleton and then propping it up with muscles is exactly what `<div onclick>` amounts to.

---

## 16. Mini Architecture Diagram

```text
HTML  (structure and meaning)
   ↓
DOM
   ↓
CSS (presentation)  +  JavaScript (behaviour)
   ↓
Rendered, accessible, indexable page
```

---

## 17. Complete Request Flow

```text
Browser requests the page
    ↓
HTML streams in and is parsed incrementally — content appears before it fully arrives
    ↓
<link rel="stylesheet"> → CSS fetched, rendering blocked until ready
    ↓
<script defer> → downloaded in parallel, executed after parsing
    ↓
DOM complete → DOMContentLoaded
    ↓
Images and fonts finish → load
    ↓
Screen readers and crawlers read the same semantic structure
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> HTML describes meaning, not appearance — choosing the semantically correct element gives you accessibility, SEO and keyboard support without writing any additional code.

---

## 19. Common Mistakes

- **`<div onclick>` instead of `<button>`**
- **Missing `alt` attributes** on images
- **Skipping heading levels** to get a particular font size
- **Inputs without labels**
- **Trusting client-side validation** as a security control
- **Div soup** — no semantic elements at all
- **`target="_blank"` without `rel="noopener"`**

---

## 20. Open Source Technologies

- **axe-core**, **Lighthouse**, **WAVE** — accessibility auditing
- **DOMPurify** — HTML sanitisation
- **W3C validator** — catch malformed markup the browser silently fixed
- **Pug**, **Handlebars**, **JSX** — templating that produces HTML

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Run Lighthouse's accessibility audit on your project and fix the top three issues.
- [ ] Navigate one of your pages using only the keyboard. Note everything you cannot reach.
- [ ] Replace one `<div>`-based control with the correct semantic element and observe what you no longer need to implement.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Server
    ↓
HTML
    ↓
DOM
    ↓
CSS + JavaScript
    ↓
Rendered page
```

## 2. Request Flow

```text
Input       an HTML document
    ↓
Processing  parsed incrementally into a semantic DOM tree
    ↓
Output      a structure browsers, screen readers and crawlers all understand
```

## 3. Real-World Usage

**Google's search ranking** uses semantic structure and Core Web Vitals directly. Correct headings, meta tags and structured data are not stylistic preferences — they determine whether content is found at all.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A markup language describing content structure and meaning |
| **Why does it exist?** | So many different tools can interpret the same document correctly |
| **Where does it belong?** | The content layer, beneath CSS and JavaScript |
| **When should I use it?** | For every web page — and always choose elements by meaning first |

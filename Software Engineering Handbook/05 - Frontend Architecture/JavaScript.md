# JavaScript

> **In one line —** the only language browsers run natively, designed in ten days, and now the most widely deployed programming language in the world.

| | |
|---|---|
| **Category** | Programming Language |
| **Architectural Layer** | Client and server |
| **Standard** | ECMAScript |
| **Related notes** | [JavaScript Engine](JavaScript%20Engine.md) · [Event Loop](Event%20Loop.md) · [TypeScript](TypeScript.md) · [Node.js Runtime](../03%20-%20Programming%20Languages%20and%20Runtime/Node.js%20Runtime.md) · [React](React.md) |

---

## 1. Short Definition

*What is it?*

JavaScript is a dynamically typed, single-threaded, interpreted-then-JIT-compiled language. It runs in every browser and, through [Node.js](../03%20-%20Programming%20Languages%20and%20Runtime/Node.js%20Runtime.md), on servers, desktops and embedded devices.

---

## 2. Purpose

*What is its main purpose?*

To make web pages interactive. Everything else it does now — servers, build tools, desktop apps, mobile apps — grew from having a monopoly on the browser.

---

## 3. Problem

*What engineering problem does it solve?*

Static documents could not respond to users. JavaScript added behaviour without a page reload, and later became the only realistic way to build applications that run everywhere without installation.

---

## 4. Architecture Position

```text
HTML  (structure)  ·  CSS  (presentation)  ·  JAVASCRIPT  (behaviour)
                                                  ↓
                                         JavaScript engine (V8)
                                                  ↓
                                            Event loop
                                                  ↓
                                    Browser APIs  /  Node APIs
```

---

## 5. The characteristics that define it

| Property | Consequence |
|---|---|
| **Dynamically typed** | Fast to write, errors surface at runtime |
| **Single-threaded** | No race conditions, but blocking freezes everything |
| **Event-driven** | Asynchronous by default via the [event loop](Event%20Loop.md) |
| **Prototype-based** | `class` is syntax over prototypes, not a separate system |
| **First-class functions** | Callbacks, closures and functional patterns everywhere |
| **Runs everywhere** | One language across the whole stack |

---

## 6. Asynchronous JavaScript

```text
CALLBACKS          fn(err, result)  →  nesting grows quickly
    ↓
PROMISES           .then().catch()  →  flat chains
    ↓
ASYNC / AWAIT      reads like synchronous code, still asynchronous underneath
```

```javascript
// All three do the same thing; the last is what you should write today
const data = await fetch(url).then(r => r.json());
```

> [!IMPORTANT]
> `await` does **not** block the thread. It suspends the current function and hands control back to the [event loop](Event%20Loop.md), which continues with other work. This is the single most misunderstood part of the language.

---

## 7. The parts that surprise people

```javascript
0.1 + 0.2 === 0.3        // false — IEEE 754 floats, same as most languages
[] + {}                  // "[object Object]" — type coercion
typeof null              // "object" — a bug from 1995, kept for compatibility
[1, 10, 2].sort()        // [1, 10, 2] — sorts as strings by default
```

> [!TIP]
> Use `===` rather than `==`, always. Loose equality applies coercion rules that almost nobody remembers correctly, and there is no case where you need it.

---

## 8. Closures — the concept worth understanding properly

```javascript
function counter() {
  let count = 0;                 // captured by the returned function
  return () => ++count;
}
const next = counter();
next(); // 1
next(); // 2
```

A function keeps access to the variables of the scope it was created in, even after that scope has returned. Closures underpin module patterns, React hooks, and most callback-based APIs.

---

## 9. Real World Example

- **Every interactive website** — there is no alternative in the browser.
- **[Node.js](../03%20-%20Programming%20Languages%20and%20Runtime/Node.js%20Runtime.md)** — backends, CLI tools, build systems.
- **React Native, Electron** — mobile and desktop applications.
- **Build tooling** — Vite, webpack and esbuild are themselves JavaScript (or increasingly Go/Rust, written for JavaScript).

---

## 10. Communication and Dependencies

- **[DOM](DOM%20and%20CSSOM.md)** — how it manipulates the page
- **Browser APIs** — `fetch`, `localStorage`, `WebSocket`, `Canvas` — none of which are part of the language
- **npm** — the package ecosystem, and a substantial part of the risk surface
- **[TypeScript](TypeScript.md)** — a typed superset that compiles to it

---

## 11. Alternatives

```text
JavaScript      universal, dynamically typed
    ↓
TypeScript      the same language with compile-time types    ← the default for new projects
    ↓
WebAssembly     compiled languages in the browser, for performance-critical code
    ↓
Elm, ReScript   stricter functional languages compiling to JS; small ecosystems
```

---

## 12. When To Use / When NOT To Use

> [!TIP]
> In the browser you have no choice. On the server, choose it for I/O-bound work and for the advantage of one language across the stack.

> [!CAUTION]
> Avoid plain JavaScript for anything beyond a small project — use [TypeScript](TypeScript.md). And avoid JavaScript entirely for CPU-heavy work, numeric computing and data science, where the ecosystem is elsewhere.

---

## 13. Advantages and Disadvantages

**Advantages**
- Runs everywhere, with no installation
- The largest package ecosystem in existence
- Fast to write and iterate on
- Excellent engines and tooling
- One language for frontend, backend and tooling

**Disadvantages**
- Dynamic typing lets whole classes of bugs reach production
- Historical design quirks that cannot be removed
- Single-threaded, so CPU work blocks everything
- npm supply-chain risk is real and recurring
- Ecosystem churn — the tooling changes constantly

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Bundle size** | Costs download, parse, compile and memory — the biggest lever |
| **Main thread** | Shared with rendering; long tasks drop frames |
| **Memory** | Garbage-collected; leaks come from retained references and listeners |
| **CPU** | Fast after JIT warm-up, but still one thread |

---

## 15. Security Considerations

> [!CAUTION]
> **XSS is JavaScript's defining vulnerability.** If an attacker can execute script in your page, they act as the logged-in user: read the DOM, steal tokens from localStorage, and issue authenticated requests.

- **Never use `eval` or `new Function`** with anything user-influenced
- **Escape output**; use `textContent` rather than `innerHTML`
- **Content Security Policy** is the strongest structural defence
- **Prototype pollution** — modifying `Object.prototype` changes behaviour application-wide
- **npm dependencies run with full privileges**; audit and pin them
- **Nothing in the browser is trustworthy** — every check must be repeated on the server

---

## 16. Mental Model

> [!NOTE]
> **JavaScript is a language that was rushed into existence and then had to carry the entire web.**
>
> It has awkward corners that cannot be fixed, because fixing them would break millions of existing pages. Modern JavaScript is what you get when very good engineering is applied on top of a foundation nobody is allowed to replace.

---

## 17. Mini Architecture Diagram

```text
Your JavaScript
    ↓
JavaScript engine (parse → bytecode → JIT)
    ↓
Event loop  (microtasks, macrotasks)
    ↓
Browser APIs (DOM, fetch)  /  Node APIs (fs, net)
    ↓
Operating system
```

---

## 18. Complete Request Flow

Handling a form submission in the browser:

```text
User clicks submit → event queued
    ↓
Event loop runs the handler
    ↓
Client-side validation (convenience only)
    ↓
await fetch('/api/orders')  → suspends; the loop continues other work
    ↓
Response arrives → continuation queued as a microtask
    ↓
Handler resumes, parses JSON
    ↓
DOM updated → rendering engine lays out and paints
    ↓
Server re-validates everything, because the client cannot be trusted
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> JavaScript is single-threaded and asynchronous — `await` suspends rather than blocks, and nothing that happens in the browser can ever be trusted by the server.

---

## 20. Common Mistakes

- **Using `==` instead of `===`**
- **Believing `await` blocks** or creates parallelism
- **Trusting client-side validation** as a security control
- **Storing tokens in localStorage**, where XSS reaches them
- **Mutating shared objects** and losing track of where a change came from
- **Adding event listeners without removing them**, leaking memory
- **`innerHTML` with user data**
- **Adding dependencies casually** without weighing the supply-chain cost

---

## 21. Open Source Technologies

- **TypeScript** — types on top of JavaScript
- **ESLint**, **Prettier** — catch mistakes and enforce style
- **Vite**, **esbuild**, **Rollup** — bundling and build tooling
- **Vitest**, **Jest**, **Playwright** — testing
- **Node.js**, **Deno**, **Bun** — runtimes outside the browser

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Write a closure-based counter and explain out loud why the variable survives.
- [ ] Take a promise chain in your code and rewrite it with `async`/`await`.
- [ ] Search your project for `eval`, `innerHTML` and `==`, and justify or fix each occurrence.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
JavaScript
    ↓
Engine (V8)
    ↓
Event loop
    ↓
Browser / Node APIs
    ↓
Operating system
```

## 2. Request Flow

```text
Input       source code plus events
    ↓
Processing  compiled by the engine, scheduled by the event loop, one thread
    ↓
Output      DOM changes, network calls, and program effects
```

## 3. Real-World Usage

JavaScript is the only language every browser runs, which is why it ended up everywhere else too. **Node.js, React Native and Electron** all exist because the pool of JavaScript developers and libraries became too large to ignore.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A dynamically typed, single-threaded, event-driven language |
| **Why does it exist?** | To make web pages interactive; everything else followed |
| **Where does it belong?** | The behaviour layer in browsers, and the runtime layer in Node |
| **When should I use it?** | Always in the browser; on the server for I/O-bound work — and prefer TypeScript |

# JavaScript Engine

> **In one line —** the program that turns JavaScript into machine code, fast, by guessing what your code will do and recompiling when it guesses wrong.

| | |
|---|---|
| **Category** | Browser / Runtime Component |
| **Architectural Layer** | Client runtime |
| **Examples** | V8 (Chrome, Node) · JavaScriptCore (Safari) · SpiderMonkey (Firefox) |
| **Related notes** | [Event Loop](Event%20Loop.md) · [Browser Internals](Browser%20Internals.md) · [JavaScript](JavaScript.md) · [Node.js Runtime](../03%20-%20Programming%20Languages%20and%20Runtime/Node.js%20Runtime.md) · [Bytecode](../03%20-%20Programming%20Languages%20and%20Runtime/Bytecode.md) |

---

## 1. Short Definition

*What is it?*

A JavaScript engine parses, compiles and executes JavaScript. Modern engines are not interpreters in any simple sense — they are **optimising JIT compilers** that produce machine code at runtime.

---

## 2. Purpose

*What is its main purpose?*

To run a dynamically typed, late-bound language at a speed close to compiled languages, and to do it inside a sandbox that untrusted code cannot escape.

---

## 3. Problem

*What engineering problem does it solve?*

JavaScript gives the engine almost nothing to work with ahead of time: no types, objects that change shape at runtime, functions that can be redefined. A naive interpreter would be far too slow for modern web applications.

---

## 4. Architecture Position

```text
Your JavaScript
    ↓
Parser → Abstract Syntax Tree
    ↓
Bytecode  (Ignition, in V8)
    ↓
Interpreter runs it, PROFILING as it goes
    ↓
Hot function detected
    ↓
Optimising compiler (TurboFan) → machine code
    ↓
Assumption violated → DEOPTIMISE → back to bytecode
    ↓
CPU
```

---

## 5. How the optimisation works

```text
function add(a, b) { return a + b; }

Called 10,000 times with numbers
    ↓
Engine observes: "always integers"
    ↓
Compiles a specialised version that adds two integers directly
    ↓
Now called once with strings
    ↓
GUARD FAILS → discard the optimised code → back to the slow path
```

> [!IMPORTANT]
> This is why **keeping object shapes and types consistent** genuinely matters for performance. An array of `{x, y}` objects with identical shapes is far faster than the same objects with fields added in different orders.

---

## 6. Hidden classes

Engines assign an internal "shape" to each object so property access can be a fixed memory offset instead of a hash lookup.

```text
FAST                              SLOW
const p = {x: 1, y: 2};           const p = {};
                                  p.x = 1;
same shape every time             p.y = 2;      ← shape changes twice
                                  delete p.x;   ← shape changes again
```

---

## 7. Garbage collection

```text
Young generation   most objects die quickly → collected often, very fast (scavenger)
    ↓  survivors promoted
Old generation     collected rarely, more expensively (mark-and-sweep, incremental)
```

Modern engines do most of this incrementally and concurrently, so pauses are short — but they are not zero, and they show up as occasional frame drops.

---

## 8. Real World Example

- **V8** powers Chrome, Edge and [Node.js](../03%20-%20Programming%20Languages%20and%20Runtime/Node.js%20Runtime.md) — the same engine on both client and server.
- **JavaScript performance on mobile** is dominated by parse and compile time, not execution: a 1 MB bundle costs real seconds on a mid-range phone before a single line runs.
- **WebAssembly** exists for code where even an excellent JIT is not enough — it ships pre-compiled and skips the guessing entirely.

---

## 9. Input, Processing, Output

**Input:** JavaScript source text.
**Processing:** parse → bytecode → interpret and profile → optimise hot paths → deoptimise on guard failure.
**Output:** program effects, plus a continuously re-tuned set of compiled functions.

---

## 10. Communication and Dependencies

- **[Event loop](Event%20Loop.md)** — schedules when JavaScript actually runs
- **Web APIs / Node APIs** — the engine itself has no `setTimeout`, no DOM, no file system; the host provides those
- **[Rendering engine](Rendering%20Engine.md)** — shares the main thread with it
- **The host environment** — a browser or Node.js

> [!TIP]
> A useful clarification: **V8 does not know what the DOM is.** The DOM, `fetch`, `setTimeout` and `console` all come from the browser, not the language. That separation is exactly why the same engine can run on a server.

---

## 11. Alternatives

```text
V8                Chrome, Edge, Node, Deno — the most widely used
JavaScriptCore    Safari, Bun
SpiderMonkey      Firefox
QuickJS           tiny, embeddable, interpreted
WebAssembly       not an alternative engine, but an alternative target
```

---

## 12. When To Use / When NOT To Use

> [!TIP]
> You do not choose the engine; you write code that either helps or hinders it. Consistent types, stable object shapes and avoiding megamorphic call sites are the levers you have.

> [!CAUTION]
> Do **not** micro-optimise for the JIT in ordinary application code. The engine is smarter than the intuition, engines change, and the real bottleneck is nearly always bundle size, network requests or layout — not arithmetic.

---

## 13. Advantages and Disadvantages

**Advantages**
- Near-native speed for a dynamically typed language
- Adapts to actual runtime behaviour, which a static compiler cannot
- Strong sandboxing — untrusted code cannot reach the OS
- Highly optimised garbage collection

**Disadvantages**
- Warm-up: the first executions are slow
- Deoptimisation makes performance unpredictable
- Parse and compile time is a real cost on mobile
- Memory-hungry — compiled code and metadata are stored

---

## 14. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Fast after warm-up; the first run of any code path is slow |
| **Memory** | Compiled code, hidden classes and inline caches all consume memory |
| **CPU** | JIT compilation and GC run alongside your code |
| **Startup** | Parsing 1 MB of JavaScript costs hundreds of milliseconds on a phone |

> [!IMPORTANT]
> **The fastest JavaScript is JavaScript you never ship.** Bundle size affects download, parse, compile and memory simultaneously — it is the highest-leverage frontend performance lever there is.

---

## 15. Security Considerations

> [!CAUTION]
> JIT compilers are a major browser attack surface. Because they generate executable machine code at runtime, a bug in the optimiser can be turned into arbitrary code execution — a large share of browser exploits target exactly this.

- **The sandbox is the real defence** — engine bugs are contained by the renderer process
- **`eval` and `new Function`** compile strings at runtime; passing user input to either is remote code execution
- **Prototype pollution** is a JavaScript-specific class where modifying `Object.prototype` changes behaviour application-wide
- **Timing side channels** led browsers to deliberately reduce timer precision after Spectre

---

## 16. Mental Model

> [!NOTE]
> **The engine is a translator who learns your accent.**
>
> At first they translate slowly, word by word. After hearing you say the same sentence a thousand times, they anticipate it and translate instantly. Then you say something unexpected — and they have to stop, unlearn the shortcut and go back to translating carefully.

---

## 17. Mini Architecture Diagram

```text
Source
    ↓
Parser → AST
    ↓
Ignition (bytecode interpreter) ──profiling──┐
    ↓                                         ↓
   run                                  TurboFan (optimiser)
    ↑                                         ↓
    └────── deoptimise ◄──── guard fails ── machine code
                                              ↓
                                             CPU
```

---

## 18. Complete Request Flow

A page loading and running JavaScript:

```text
Script downloaded  (network)
    ↓
Parsed                              ← costs real time on large bundles
    ↓
Compiled to bytecode
    ↓
Executed on the MAIN THREAD — rendering is blocked while this runs
    ↓
Hot functions promoted to optimised machine code
    ↓
Objects allocated → young generation → GC
    ↓
Page becomes interactive
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> Modern JavaScript engines compile to machine code by guessing types from observed behaviour — so consistent shapes help, and shipping less code helps far more.

---

## 20. Common Mistakes

- **Shipping huge bundles** and blaming slow devices
- **Changing object shapes** after creation in hot code
- **Using `eval` or `new Function`** with anything user-influenced
- **Micro-optimising** instead of measuring the real bottleneck
- **Benchmarking without warm-up**, measuring the interpreter rather than the optimised code
- **Assuming the engine provides the DOM** — it does not

---

## 21. Open Source Technologies

- **V8**, **SpiderMonkey**, **JavaScriptCore** — the engines themselves
- **WebAssembly** — a compiled target for performance-critical code
- **esbuild**, **Vite**, **Rollup** — reduce what you ship in the first place
- **Chrome DevTools Performance panel**, **`--prof`** in Node — see where time goes

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Measure your production bundle size and estimate its parse cost on a mid-range phone.
- [ ] Write a function called with consistent types, then with mixed types, and compare timings.
- [ ] Explain in two sentences why V8 can run both in Chrome and in Node.js.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
JavaScript source
    ↓
Parser → bytecode → JIT
    ↓
Machine code
    ↓
CPU
```

## 2. Request Flow

```text
Input       JavaScript source text
    ↓
Processing  parse, compile to bytecode, profile, optimise, deoptimise as needed
    ↓
Output      program effects, executed at close to native speed
```

## 3. Real-World Usage

**Node.js** exists because V8 was extracted from Chrome and paired with an I/O library. A browser component became a server platform purely because the engine was cleanly separable from the browser around it.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An optimising JIT compiler and runtime for JavaScript |
| **Why does it exist?** | Because a dynamically typed language needs runtime information to be fast |
| **Where does it belong?** | Inside the browser's renderer process, or inside Node.js |
| **When should I use it?** | Understand it to reason about bundle size, warm-up and JS performance |

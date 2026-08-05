# Browser Internals

> **In one line —** a browser is an operating system pretending to be an application: it fetches, parses, sandboxes, executes and paints — all in about 16 milliseconds per frame.

| | |
|---|---|
| **Category** | Overview note *(hub for this section)* |
| **Architectural Layer** | Client |
| **Sub-topics** | [Rendering Engine](Rendering%20Engine.md) · [JavaScript Engine](JavaScript%20Engine.md) · [Event Loop](Event%20Loop.md) · [DOM and CSSOM](DOM%20and%20CSSOM.md) |

---

## 1. Short Definition

A browser is a program that downloads untrusted code from strangers and runs it safely on your machine. Everything about its architecture follows from that sentence.

---

## 2. The main components

| Component | Job |
|---|---|
| **Networking** | DNS, TCP, TLS, HTTP, caching |
| **[Rendering engine](Rendering%20Engine.md)** | Parse HTML/CSS, lay out, paint (Blink, WebKit, Gecko) |
| **[JavaScript engine](JavaScript%20Engine.md)** | Parse and execute JS (V8, JavaScriptCore, SpiderMonkey) |
| **[Event loop](Event%20Loop.md)** | Coordinate tasks, rendering, callbacks |
| **Storage** | Cookies, localStorage, IndexedDB, cache |
| **Sandbox** | Keep pages away from your operating system |

---

## 3. Process architecture

Modern browsers are **multi-process**, and for security reasons rather than performance ones.

```text
┌─────────────── BROWSER PROCESS ───────────────┐
│  UI, address bar, tabs, network, storage       │
└───────┬──────────────┬──────────────┬──────────┘
        ↓              ↓              ↓
  RENDERER        RENDERER        RENDERER        ← one per site, sandboxed
  (tab A)         (tab B)         (tab C)
        ↓
  GPU PROCESS  — compositing and painting
```

> [!IMPORTANT]
> **Site isolation** puts different sites in different processes. A compromised page can only reach its own process, and Spectre-class attacks cannot read another site's data because that data is not in the same address space.

---

## 4. What happens when you open a page

```text
1. URL entered
    ↓
2. DNS → TCP → TLS → HTTP GET                  ← see folder 04
    ↓
3. HTML arrives and is parsed → DOM
    ↓
4. CSS discovered and parsed → CSSOM           ← blocks rendering
    ↓
5. JS discovered → downloaded and executed     ← blocks parsing unless async/defer
    ↓
6. DOM + CSSOM → Render Tree
    ↓
7. LAYOUT   — where does everything go?
    ↓
8. PAINT    — what colour is each pixel?
    ↓
9. COMPOSITE — assemble layers on the GPU
    ↓
10. Repeat 7–9 for every change, ~60 times per second
```

---

## 5. The 16 millisecond budget

At 60 frames per second, the browser has **16.7 ms** to do everything for one frame.

```text
JavaScript  →  Style  →  Layout  →  Paint  →  Composite
     ────────────── all of this in 16 ms ──────────────
```

> [!CAUTION]
> A single long JavaScript task blocks all of it. 100 ms of synchronous work means six dropped frames and visibly janky scrolling — the [event loop](Event%20Loop.md) note explains exactly why.

---

## 6. Why this matters to a backend developer

- **Render-blocking resources** — CSS in `<head>` blocks painting; a `<script>` without `defer` blocks parsing
- **Time to First Byte** is yours; everything after it is the browser's
- **Server-side rendering** exists because step 5 above is slow on weak devices
- **CORS, cookies and CSP** are enforced here, and misconfiguring them on the server breaks the client
- **Caching headers** you set determine whether steps 2–5 happen at all

---

## 7. Storage in the browser

| Mechanism | Size | Sent to server? | Use for |
|---|---|---|---|
| **Cookies** | ~4 KB | Yes, every request | Session identifiers |
| **localStorage** | ~5–10 MB | No | Preferences, non-sensitive cache |
| **sessionStorage** | ~5–10 MB | No | Per-tab temporary state |
| **IndexedDB** | Large | No | Offline data, structured storage |
| **Cache API** | Large | No | Service worker asset caching |

> [!CAUTION]
> **localStorage is readable by any JavaScript on the page**, including anything injected via XSS. Storing an authentication token there means one XSS vulnerability equals full account takeover. `HttpOnly` cookies are not readable by JavaScript, which is why they remain the safer default for session tokens.

---

## 8. The security model

- **Same-Origin Policy** — a page from origin A cannot read data from origin B. This is the foundational rule of web security.
- **CORS** — the server's way of granting selective exceptions to that rule
- **Content Security Policy** — restricts which scripts may run at all, the strongest defence against XSS
- **Sandboxing** — renderer processes cannot touch the file system or the OS
- **Site isolation** — separate processes per site

---

## 9. Mental Model

> [!NOTE]
> **A browser is an airport.**
>
> Everything arriving from outside is treated as untrusted and goes through security ([sandbox](JavaScript%20Engine.md)). Different airlines are kept in separate terminals (site isolation). Everything runs on a strict timetable — miss your 16 ms slot and passengers notice.

---

## 10. Key Takeaway

> [!IMPORTANT]
> A browser downloads and runs untrusted code from strangers, safely, sixty times a second — every architectural decision inside it exists to serve security or that frame budget.

---

## 11. Common Mistakes

- **Blocking the main thread** with long synchronous JavaScript
- **Storing auth tokens in localStorage**
- **Loading render-blocking scripts** without `defer` or `async`
- **Assuming all users have your laptop** — mid-range phones parse JavaScript several times slower
- **Ignoring the cost of large bundles** — download, parse, compile and execute all cost time

---

## 12. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 13. Workbook Exercise

- [ ] Open DevTools → Performance, record a page load, and identify layout and paint phases.
- [ ] Check whether your app stores any authentication token in localStorage.
- [ ] Throttle the CPU to 4× slowdown in DevTools and load your own site.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
URL
    ↓
Network (DNS, TCP, TLS, HTTP)
    ↓
HTML → DOM   +   CSS → CSSOM
    ↓
Render tree → Layout → Paint → Composite
    ↓
Pixels on screen
```

## 2. Request Flow

```text
Input       a URL
    ↓
Processing  fetch, parse, execute, lay out, paint, composite
    ↓
Output      rendered pixels, updated ~60 times per second
```

## 3. Real-World Usage

**Chrome's site isolation** was accelerated by the Spectre vulnerability. Once it became clear that JavaScript could read memory it was never granted access to, putting each site in its own process stopped being an optimisation and became a requirement.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A sandboxed runtime that fetches, parses, executes and renders web content |
| **Why does it exist?** | To run untrusted code from anywhere without endangering the machine |
| **Where does it belong?** | Entirely on the client, opposite your server |
| **When should I use it?** | Understand it whenever frontend performance or web security matters |

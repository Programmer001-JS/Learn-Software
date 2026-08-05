# Node.js Runtime

> **In one line —** JavaScript outside the browser: one thread, an event loop, and non-blocking I/O — which makes it excellent at waiting and poor at computing.

| | |
|---|---|
| **Category** | Language Runtime |
| **Architectural Layer** | Runtime |
| **Built from** | V8 (JavaScript engine) + libuv (async I/O) |
| **Related notes** | [Event Loop](../05%20-%20Frontend%20Architecture/Event%20Loop.md) · [JavaScript Engine](../05%20-%20Frontend%20Architecture/JavaScript%20Engine.md) · [Runtime Explained](Runtime%20Explained.md) · [Express.js](../06%20-%20Backend%20Architecture/Express.js.md) · [NestJS](../06%20-%20Backend%20Architecture/NestJS.md) |

---

## 1. Short Definition

*What is it?*

Node.js is a runtime that executes JavaScript on a server. It combines Google's **V8** engine (which compiles and runs JavaScript) with **libuv** (which provides an [event loop](../05%20-%20Frontend%20Architecture/Event%20Loop.md) and non-blocking I/O), plus a standard library for files, networking and processes.

---

## 2. Purpose

*What is its main purpose?*

To handle a very large number of concurrent I/O operations on a single thread, and to let one language be used on both sides of a web application.

---

## 3. Problem

*What engineering problem does it solve?*

Traditional servers dedicate a [thread](../02%20-%20Computer%20Science%20Fundamentals/Thread.md) per connection. Each thread costs megabytes of stack and a [context switch](../02%20-%20Computer%20Science%20Fundamentals/Context%20Switching.md) whenever it blocks. Ten thousand connections means ten thousand mostly-idle threads.

```text
THREAD PER CONNECTION          NODE.JS

10,000 threads                 1 thread + an event loop
~10 GB of stacks               ~50 MB
constant context switching     no switching
mostly blocked, waiting        never blocks, just registers callbacks
```

---

## 4. Architecture Position

```text
Your JavaScript / TypeScript
    ↓
Framework  (Express, NestJS, Fastify)
    ↓
Node.js standard library
    ↓
┌──────── NODE.JS RUNTIME ────────┐
│  V8: parse → JIT → machine code │
│  libuv: EVENT LOOP + thread pool│
└────────────────┬─────────────────┘
                 ↓
     Operating system (epoll / kqueue)
                 ↓
             Hardware
```

---

## 5. The event loop — the whole idea

> [!IMPORTANT]
> Your JavaScript runs on **one thread**. When it starts an I/O operation, Node registers it with the OS and immediately moves on. When the OS reports completion, the callback is queued and runs when the thread is next free.

```text
Request A arrives → start a database query → register a callback → MOVE ON
Request B arrives → start an HTTP call     → register a callback → MOVE ON
Request C arrives → CPU work               → ⚠ BLOCKS EVERYTHING
    ↓
A's query completes → its callback runs
B's call completes  → its callback runs
```

The consequence follows directly: **any long synchronous computation freezes every other request in the process.**

---

## 6. Input and Output

**Input:** JavaScript or compiled TypeScript, plus `node_modules` dependencies and environment configuration.

**Output:** a single-threaded process capable of thousands of concurrent connections — with all CPU work serialised through one thread.

---

## 7. Internal Idea

*How does it work internally?*

**V8** compiles JavaScript straight to machine code with a tiered JIT (Ignition interpreter, then TurboFan optimiser), specialising on observed types and deoptimising when assumptions break.

**libuv** provides the event loop and a small thread pool (4 threads by default) for operations the OS cannot do asynchronously — file system work, DNS, compression, crypto.

> [!TIP]
> "Node is single-threaded" is a useful simplification but not literally true. *Your JavaScript* is single-threaded; libuv's pool and V8's background compilation use other threads.

---

## 8. Communication

- **Operating system** — epoll on Linux, kqueue on BSD/macOS
- **Native addons** — C++ modules through N-API
- **Worker threads** — separate V8 isolates for CPU work, communicating by message passing
- **Cluster / PM2** — multiple Node processes to use multiple cores

---

## 9. Dependencies

- A specific Node version — LTS versions are the sane default
- `node_modules`, which is frequently enormous and is part of your deployment
- For CPU parallelism, multiple processes or worker threads

---

## 10. Alternatives

```text
Node.js     the default; largest ecosystem
    ↓
Deno        TypeScript-first, secure by default, permission-based
    ↓
Bun         much faster startup and package installs; younger ecosystem
    ↓
Go          real concurrency plus real parallelism, compiled, small binary
    ↓
Python asyncio   the same event-loop model in a different ecosystem
```

---

## 11. When To Use

> [!TIP]
> Use Node for **I/O-bound** services: REST and GraphQL APIs, gateways, WebSocket servers, anything that mostly waits on databases and other services. Sharing one language across frontend and backend is a genuine team-level advantage.

---

## 12. When NOT To Use

> [!CAUTION]
> - **CPU-bound work** — image processing, encryption, large data transformation. One long loop blocks every concurrent request in that process.
> - **Heavy numeric or scientific computing** — the ecosystem is in Python.
> - **Where strict typing and long-term maintainability matter most** — TypeScript helps considerably, but the runtime still erases types.

---

## 13. Advantages

- Excellent concurrency for I/O with very low memory per connection
- One language across the whole stack
- The largest package ecosystem in existence (npm)
- Fast startup — good for serverless
- V8 is genuinely fast for a dynamic language

---

## 14. Disadvantages

- A single blocking operation stalls the entire process
- No CPU parallelism without worker threads or multiple processes
- Callback and promise-based error handling is easy to get wrong
- `node_modules` size and supply-chain risk are real operational concerns
- Dynamic typing at runtime, whatever TypeScript promised at compile time

---

## 15. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Very good for I/O; poor for CPU-heavy work |
| **Memory** | ~50 MB baseline; a few KB per connection rather than a few MB |
| **CPU** | One core per process for your JavaScript |
| **Startup** | ~50 ms — well suited to serverless |
| **Container** | 100–500 MB, dominated by `node_modules` |

---

## 16. Security Considerations

> [!CAUTION]
> **npm is your largest attack surface.** A typical project pulls in hundreds of transitive dependencies, each running with your process's full privileges. Real supply-chain compromises (`event-stream`, `ua-parser-js`) have shipped malicious code to thousands of applications this way.

- Use `npm audit`, lockfiles and pinned versions; review what you add
- **Prototype pollution** is a JavaScript-specific vulnerability class worth understanding
- Never pass user input to `eval` or `new Function`
- One unhandled promise rejection can terminate the process — and with it every in-flight request

---

## 17. Mental Model

> [!NOTE]
> **Node.js is a single waiter in a very busy restaurant.**
>
> They never stand at a table waiting for the kitchen — they take an order, pass it through, and move to the next table. One waiter serves a hundred tables comfortably. But the moment they sit down to peel potatoes themselves (CPU work), every table waits.

---

## 18. Mini Architecture Diagram

```text
Incoming connections
    ↓
┌──────── EVENT LOOP (one thread) ────────┐
│  timers → pending → poll → check → close│
└────┬────────────────────────────┬────────┘
     ↓                            ↓
 Callback queue           libuv thread pool (4)
     ↓                            ↓
Your JavaScript            file I/O, DNS, crypto
     ↓
 V8 → machine code → CPU
```

---

## 19. Complete Request Flow

```text
HTTP request arrives
    ↓
Event loop picks it up, runs your handler
    ↓
Handler calls the database → request registered with the OS → RETURNS IMMEDIATELY
    ↓
Event loop serves other requests meanwhile
    ↓
Database responds → OS notifies libuv → callback queued
    ↓
Event loop runs the callback → response serialised → written to the socket
```

And the failure mode to remember:

```text
Handler runs a 2-second synchronous loop
    ↓
Event loop is BLOCKED
    ↓
Every other request waits 2 seconds
    ↓
Health checks time out; the container is restarted
```

---

## 20. Key Takeaway

> [!IMPORTANT]
> Node.js is superb at waiting and poor at computing — never block the event loop, and use processes or worker threads for CPU work.

---

## 21. Common Mistakes

- **Blocking the event loop** with synchronous or CPU-heavy code
- **Using synchronous file APIs** (`readFileSync`) in request handlers
- **Unhandled promise rejections**, which crash the process
- **Running one Node process on a multi-core machine** — use cluster or a process manager
- **Adding dependencies casually** without considering the supply chain
- **Assuming TypeScript gives runtime type safety** — validate input at the boundary

---

## 22. Open Source Technologies

- **Node.js**, **Deno**, **Bun** — the runtimes
- **Express**, **Fastify**, **NestJS** — web frameworks
- **PM2**, **cluster** — multi-process management
- **clinic.js**, **0x**, `--prof` — profiling and event-loop diagnostics

---

## 23. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 24. Workbook Exercise

- [ ] Add a synchronous 2-second loop to an endpoint and observe what happens to concurrent requests.
- [ ] Count the total transitive dependencies in one of your projects (`npm ls --all | wc -l`).
- [ ] Explain in two sentences why Node handles 10,000 connections better than a thread-per-connection server.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
JavaScript / TypeScript
    ↓
Framework
    ↓
Node.js (V8 + libuv event loop)
    ↓
Operating system (epoll)
    ↓
Hardware
```

## 2. Request Flow

```text
Input       an HTTP request or other I/O event
    ↓
Processing  handled on one thread; I/O registered with the OS and awaited via callbacks
    ↓
Output      a response, with the thread never blocked while waiting
```

## 3. Real-World Usage

**Netflix** moved its user-facing web layer to Node.js and reported large improvements in startup time and developer velocity. That layer is almost entirely I/O — assembling data from backend services — which is exactly the workload the event loop is built for.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A server-side JavaScript runtime built on V8 and an event loop |
| **Why does it exist?** | Because thread-per-connection does not scale to tens of thousands of connections |
| **Where does it belong?** | Between JavaScript application code and the operating system |
| **When should I use it?** | I/O-bound services — never for CPU-heavy work on the main thread |

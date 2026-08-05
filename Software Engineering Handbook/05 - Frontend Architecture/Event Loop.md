# Event Loop

> **In one line —** JavaScript has one thread; the event loop is the rule that decides what that thread does next, and why blocking it freezes everything.

| | |
|---|---|
| **Category** | Runtime Mechanism |
| **Architectural Layer** | Client runtime / Node.js |
| **Related notes** | [JavaScript Engine](JavaScript%20Engine.md) · [Browser Internals](Browser%20Internals.md) · [Node.js Runtime](../03%20-%20Programming%20Languages%20and%20Runtime/Node.js%20Runtime.md) · [Thread](../02%20-%20Computer%20Science%20Fundamentals/Thread.md) · [Scheduler](../02%20-%20Computer%20Science%20Fundamentals/Scheduler.md) |

---

## 1. Short Definition

*What is it?*

The event loop is a loop that repeatedly asks: *is there work to do?* It takes one task at a time from a queue, runs it **to completion**, then takes the next. There is exactly one thread running your JavaScript.

---

## 2. Purpose

*What is its main purpose?*

To let a single thread handle thousands of concurrent operations without blocking — by never waiting for anything, and instead being notified when results are ready.

---

## 3. Problem

*What engineering problem does it solve?*

A browser must stay responsive while fetching data, and a server must handle many connections. The classic answer is one [thread](../02%20-%20Computer%20Science%20Fundamentals/Thread.md) per task, which costs megabytes of stack and constant [context switching](../02%20-%20Computer%20Science%20Fundamentals/Context%20Switching.md).

```text
THREADS                       EVENT LOOP
one per task                  one thread total
blocked while waiting         never blocks; registers a callback and moves on
MBs of memory each            KBs per pending operation
context switch overhead       none
```

---

## 4. Architecture Position

```text
Your JavaScript
    ↓
┌──────────── EVENT LOOP ────────────┐
│  call stack                        │
│  microtask queue  (promises)       │
│  macrotask queue  (timers, I/O)    │
└──────────────┬─────────────────────┘
               ↓
   Web APIs (browser)  /  libuv (Node)
               ↓
       Operating system
```

> [!IMPORTANT]
> `setTimeout`, `fetch` and file I/O are **not** part of JavaScript. The host environment provides them and runs them elsewhere; the engine only receives the callback afterwards.

---

## 5. How one turn works

```text
1. Take the next MACROTASK, run it to completion
    ↓
2. Drain the ENTIRE microtask queue     ← all promises, before anything else
    ↓
3. (Browser) render a frame if it is time
    ↓
4. Repeat
```

| Queue | Contains | Priority |
|---|---|---|
| **Microtasks** | `Promise.then`, `await`, `queueMicrotask` | **Higher** — drained completely each turn |
| **Macrotasks** | `setTimeout`, `setInterval`, I/O, events | Lower — one per turn |

---

## 6. The classic ordering example

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
console.log('4');

// Output: 1, 4, 3, 2
```

```text
'1'  → runs immediately, synchronous
setTimeout → macrotask queue
Promise    → microtask queue
'4'  → runs immediately, synchronous
    ↓ call stack empty
Microtasks drained first  → '3'
    ↓
Next macrotask            → '2'
```

> [!TIP]
> `setTimeout(fn, 0)` does not mean "run now". It means "run on a future turn of the loop, after all pending microtasks."

---

## 7. Blocking — the failure mode that matters

```javascript
// This freezes the entire page for 3 seconds
const end = Date.now() + 3000;
while (Date.now() < end) {}
```

```text
Long synchronous task running
    ↓
Event loop cannot proceed
    ↓
No clicks handled, no animations, no rendering, no network callbacks
    ↓
Browser: page frozen        Node: every concurrent request stalled
```

> [!CAUTION]
> In Node.js this is worse than in a browser: one blocking operation stalls **every user's request** in that process, not just one page. It is the single most important thing to avoid in Node.

---

## 8. Real World Example

- **Node.js** handles tens of thousands of concurrent connections on one thread precisely because it never waits.
- **A janky scroll** almost always means a long task blocked the loop past the 16 ms frame budget.
- **Web Workers** exist to move CPU work off this thread — they are separate threads that communicate by message passing and cannot touch the DOM.
- **`async`/`await`** is syntax over promises, which are microtasks; it does not create threads.

---

## 9. Input, Processing, Output

**Input:** tasks — user events, timers, completed I/O, resolved promises.
**Processing:** one macrotask, then all microtasks, then possibly a render, repeatedly.
**Output:** callbacks executed in a defined order, on one thread.

---

## 10. Communication and Dependencies

- **[JavaScript engine](JavaScript%20Engine.md)** — executes each task
- **Web APIs / libuv** — perform the actual asynchronous work elsewhere
- **[Rendering engine](Rendering%20Engine.md)** — shares the same thread; a blocked loop means no frames
- **Operating system** — `epoll`/`kqueue` notify when I/O is ready

---

## 11. Alternatives

```text
Event loop        one thread, huge I/O concurrency, no locks, no races
    ↓
Thread pool       real parallelism, but locks, races and memory cost
    ↓
Goroutines        user-space scheduling: concurrency AND parallelism
    ↓
Web Workers       separate threads for CPU work, message-passing only
```

---

## 12. When To Use

> [!TIP]
> The event loop model is ideal for **I/O-bound** work: APIs, gateways, UI event handling, WebSocket servers. Anything that mostly waits.

---

## 13. When NOT To Use

> [!CAUTION]
> Never do CPU-heavy work on the loop: image processing, large sorts, encryption, big JSON parsing, or synchronous file reads. Move it to a **Web Worker** (browser), a **worker thread** or a separate **process** (Node), or a background job queue.

---

## 14. Advantages and Disadvantages

**Advantages**
- Massive I/O concurrency with minimal memory
- No locks, no race conditions on shared state
- Simple mental model once understood
- No context-switching overhead

**Disadvantages**
- One blocking operation stalls everything
- No CPU parallelism without extra processes or workers
- Execution order is subtle — microtask vs macrotask trips up most developers
- Deep async chains make stack traces harder to read

---

## 15. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Excellent for I/O; poor for CPU |
| **Memory** | A few KB per pending operation, versus MBs per thread |
| **CPU** | One core per event loop — use cluster/workers for more |
| **Latency** | Any task over ~50 ms is user-visible; over 16 ms drops frames |

---

## 16. Security Considerations

> [!CAUTION]
> **Blocking the event loop is a denial-of-service vector.** If any user-supplied input can cause a long synchronous operation — a catastrophic regular expression, an enormous JSON body, an unbounded loop — a single request can freeze the whole process.

- **ReDoS** — a badly written regex on hostile input can hang the loop indefinitely
- **Limit request body sizes** before parsing
- **Unhandled promise rejections** terminate a Node process by default, taking every in-flight request with it

---

## 17. Mental Model

> [!NOTE]
> **The event loop is a single waiter in a busy restaurant.**
>
> They take an order and pass it to the kitchen rather than standing there waiting — so one waiter serves fifty tables. But if they sit down to peel potatoes themselves, every table waits. **Microtasks are the urgent notes they check between every table; macrotasks are the tables themselves.**

---

## 18. Mini Architecture Diagram

```text
        ┌─────────── CALL STACK ───────────┐
        │  currently executing function     │
        └───────────────┬───────────────────┘
                        ↑  (empty?)
        ┌───────────────┴──────────────────┐
        │  1. drain ALL microtasks          │  promises
        │  2. render (browser)              │
        │  3. take ONE macrotask            │  timers, I/O, events
        └───────────────┬──────────────────┘
                        ↑
             Web APIs / libuv thread pool
                        ↑
                Operating system
```

---

## 19. Complete Request Flow

A Node.js API request:

```text
Request arrives → macrotask queued
    ↓
Event loop picks it up, runs the handler
    ↓
await db.query()  → registered with the OS, handler SUSPENDS, loop moves on
    ↓
Loop serves other requests meanwhile      ← this is the whole benefit
    ↓
Database responds → OS notifies libuv → continuation queued as a MICROTASK
    ↓
Loop drains microtasks → handler resumes
    ↓
Response written to the socket
```

---

## 20. Key Takeaway

> [!IMPORTANT]
> One thread runs all your JavaScript; the event loop keeps it busy by never waiting — so anything that makes it wait, or compute for long, freezes everything.

---

## 21. Common Mistakes

- **CPU-heavy work on the main thread**
- **Synchronous APIs** (`readFileSync`, `execSync`) inside request handlers
- **Assuming `setTimeout(fn, 0)` runs immediately**
- **Confusing async with parallel** — `async` does not create threads
- **Unhandled promise rejections**
- **Running one Node process on a multi-core machine**
- **Unbounded regexes on user input** — ReDoS

---

## 22. Open Source Technologies

- **libuv** — the event loop implementation behind Node.js
- **Web Workers**, **worker_threads** — move CPU work off the loop
- **clinic.js**, **`--trace-event-categories`** — diagnose event loop lag
- **Chrome DevTools Performance panel** — see long tasks visually

---

## 23. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 24. Workbook Exercise

- [ ] Predict the output of the four-line ordering example, then run it and check.
- [ ] Add a 2-second synchronous loop to an endpoint and observe what happens to other requests.
- [ ] Find one synchronous call in a request path of yours and make it asynchronous.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
JavaScript
    ↓
Event loop (call stack, microtasks, macrotasks)
    ↓
Web APIs / libuv
    ↓
Operating system
```

## 2. Request Flow

```text
Input       events, timers, completed I/O, resolved promises
    ↓
Processing  one macrotask → all microtasks → optional render → repeat
    ↓
Output      callbacks executed in a defined order on a single thread
```

## 3. Real-World Usage

**Node.js** made this model mainstream on the server. A single-threaded process handling tens of thousands of connections was counterintuitive in 2009 and is now the default architecture for I/O-heavy services.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The mechanism deciding what JavaScript's single thread does next |
| **Why does it exist?** | To achieve high concurrency without threads |
| **Where does it belong?** | Between your code and the host's asynchronous APIs |
| **When should I use it?** | For I/O-bound work — never for CPU-bound work on the main thread |

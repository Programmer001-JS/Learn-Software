# Multithreading

> **In one line —** running several threads at once inside one program: real speed for CPU work, and a whole category of bugs that only appear under load.

| | |
|---|---|
| **Category** | Programming Model |
| **Architectural Layer** | User Space |
| **Related notes** | [Thread](Thread.md) · [Process](Process.md) · [Context Switching](Context%20Switching.md) · [Scheduler](Scheduler.md) · [Background Workers](../10%20-%20Distributed%20Systems/Background%20Workers.md) |

---

## 1. Short Definition

*What is it?*

Multithreading is designing a program so that several [threads](Thread.md) execute concurrently inside one [process](Process.md), sharing memory. On a multi-core machine, several of them genuinely run at the same instant.

---

## 2. Purpose

*What is its main purpose?*

To use more than one CPU core within a single program, and to avoid one slow operation blocking everything else.

---

## 3. Problem

*What engineering problem does it solve?*

CPU clock speeds stopped increasing around 2005; manufacturers added cores instead. A single-threaded program uses **one** core no matter how many the machine has.

```text
16-core server, single-threaded program
    ↓
15 cores idle, 6% of the machine used
```

---

## 4. Architecture Position

```text
Application
    ↓
Thread pool  (a bounded number of workers)
    ↓
Threads: T1  T2  T3  T4
    ↓
Scheduler
    ↓
Core 0  Core 1  Core 2  Core 3     ← genuine parallelism
```

---

## 5. Real World Example

- **Video encoding** splits a file into segments, encodes each on its own thread, and reassembles them — nearly linear speedup with core count.
- **Java / .NET web servers** handle each request on a pooled thread.
- **Image processing** divides an image into tiles processed in parallel.
- **Databases** use multiple threads for parallel query execution, background writing and vacuuming.

---

## 6. Input

*What does it receive as input?*

Work that can be split into pieces which are largely **independent** of one another. Independence is the entire precondition — if the pieces constantly need the same shared data, you will spend more time locking than working.

---

## 7. Processing

*What happens inside it?*

```text
Split the work
    ↓
Hand each piece to a thread
    ↓
Threads run concurrently, coordinating on shared state via locks
    ↓
Join / collect results
```

---

## 8. Output

*What does it return?*

The combined result, ideally in a fraction of the single-threaded time. In practice never a full 1/N, because of coordination overhead.

---

## 9. Internal Idea — Amdahl's Law

*How does it work internally?*

The part of a program that **cannot** be parallelised sets a hard ceiling on the achievable speedup.

```text
If 10% of the work is inherently sequential:
    ↓
Maximum speedup is 10×, even with 1,000 cores
```

> [!IMPORTANT]
> Adding threads has diminishing returns very quickly. Doubling the threads almost never halves the time, and past a certain point it makes things slower.

---

## 10. Communication

- **Shared memory** — fast, unsafe without discipline
- **Mutex / lock** — one thread in a critical section at a time
- **Read-write lock** — many readers or one writer
- **Atomic operations** — lock-free counters and flags
- **Concurrent queues** — the safest pattern: pass work, do not share state

---

## 11. Dependencies

- A language and runtime with real threading support
- Work that genuinely decomposes into independent pieces
- Multiple CPU cores, for any actual speedup

---

## 12. Alternatives

```text
Multithreading    shared memory, real parallelism, race conditions
    ↓
Multiprocessing   isolated, no races, higher memory cost — required in Python for CPU work
    ↓
Async / event loop  one thread, thousands of I/O tasks, no locks
    ↓
Distributed       spread across machines when one is not enough
```

| Work type | Best model |
|---|---|
| CPU-bound, no GIL | Multithreading |
| CPU-bound, Python | Multiprocessing |
| I/O-bound | Async / event loop |
| Beyond one machine | Distributed workers + a queue |

---

## 13. When To Use

> [!TIP]
> Use multithreading when work is CPU-bound, splits into independent pieces, and your language has no GIL. Otherwise reach for async or separate processes.

---

## 14. When NOT To Use

> [!CAUTION]
> - **I/O-bound work** — the threads will simply wait in parallel while consuming memory
> - **Python CPU-bound work** — the GIL blocks it; use `multiprocessing`
> - **Heavily shared mutable state** — the locking will eat the gains
> - **When single-threaded is fast enough** — you are buying a whole class of non-reproducible bugs for a speedup you did not need

---

## 15. Advantages

- Uses all available cores
- Shared memory means no serialisation cost between workers
- Cheaper than processes in both memory and startup
- Keeps applications responsive during long operations

---

## 16. Disadvantages

- **Race conditions** — intermittent, load-dependent, hard to reproduce
- **Deadlocks** — mutual waiting that hangs the program entirely
- Debugging is genuinely difficult; a debugger changes the timing that caused the bug
- Locks can serialise everything, eliminating the benefit
- Amdahl's Law caps the payoff regardless of effort

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Up to N× for parallel work, limited by the sequential fraction |
| **Memory** | 1–8 MB of stack per thread |
| **CPU** | Excess threads cause switching overhead, not throughput |
| **GPU** | For massively parallel maths, a GPU beats any number of CPU threads |
| **Network** | Irrelevant to the bottleneck if you are waiting on I/O |

> [!TIP]
> A reasonable default for CPU-bound work is **threads ≈ number of cores**. For I/O-bound work with threads, higher counts help — but async helps far more.

---

## 18. Security Considerations

> [!CAUTION]
> **TOCTOU** (time-of-check to time-of-use) races are exploitable: one thread checks a permission, another changes the state, and the first acts on a stale answer. Security checks and the actions they guard must be atomic.

- All threads share the process's credentials and memory — there is no boundary between them
- Race conditions in payment, inventory or balance logic cause real financial loss; use database transactions rather than in-memory locks for anything that matters
- Thread-unsafe libraries used from multiple threads corrupt data silently

---

## 19. Mental Model

> [!NOTE]
> **Multithreading is four cooks in one kitchen.**
>
> Four cooks can prepare a meal much faster than one — until two of them reach for the same pan. Coordination (locks) prevents disaster and also slows everyone down, and adding a twentieth cook to a small kitchen makes the meal take longer, not shorter.

---

## 20. Mini Architecture Diagram

```text
Task
    ↓
Split into independent pieces
    ↓
┌────┬────┬────┬────┐
│ T1 │ T2 │ T3 │ T4 │   thread pool
└─┬──┴─┬──┴─┬──┴─┬──┘
  ↓    ↓    ↓    ↓
Core0 Core1 Core2 Core3
  ↓    ↓    ↓    ↓
    Join results
        ↓
   Final output
```

---

## 21. Complete Request Flow

Processing an uploaded image in parallel:

```text
Upload received
    ↓
Split into 4 tiles
    ↓
Submit 4 tasks to the thread pool
    ↓
T1 T2 T3 T4 process concurrently on 4 cores
    ↓
Each writes to its own output buffer   ← no sharing, no locks needed
    ↓
Main thread joins, assembles the tiles
    ↓
Response
```

> [!IMPORTANT]
> The reason this scales well is that each thread writes to a **separate** buffer. Design for independence first; reach for locks only when it is genuinely unavoidable.

---

## 22. Key Takeaway

> [!IMPORTANT]
> Multithreading turns extra cores into speed only when the work splits into genuinely independent pieces — otherwise the locks give back everything you gained.

---

## 23. Common Mistakes

- **Using threads for I/O-bound work** where async would be simpler and better
- **Unsynchronised shared state** — races that pass every test and fail in production
- **Locking too broadly** and serialising everything
- **Inconsistent lock ordering**, producing deadlocks
- **Unbounded thread creation** instead of a bounded pool
- **Expecting linear speedup** and ignoring Amdahl's Law
- **In-memory locks for business invariants** that need database transactions

---

## 24. Open Source Technologies

- **Java concurrency utilities**, **.NET TPL** — mature thread pools and primitives
- **OpenMP**, **Intel TBB** — parallel loops in C/C++
- **Rayon** (Rust) — data parallelism with compile-time safety
- **Go goroutines**, **Tokio**, **asyncio** — lighter-weight concurrency models
- **ThreadSanitizer** — detect race conditions before production does

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Take a CPU-heavy loop in your project, parallelise it, and measure the actual speedup against the number of cores.
- [ ] Estimate the sequential fraction of that task and predict the ceiling from Amdahl's Law before measuring.
- [ ] Find one piece of shared mutable state in your code and decide how it should be protected.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Application
    ↓
Thread pool
    ↓
Scheduler
    ↓
Multiple CPU cores
```

## 2. Request Flow

```text
Input       work that decomposes into independent pieces
    ↓
Processing  threads run in parallel, coordinating through locks or queues
    ↓
Output      combined result, faster by less than N×
```

## 3. Real-World Usage

**ffmpeg** encodes video by splitting it across threads, which is why encoding time drops sharply on a many-core machine. It also illustrates the limit: the muxing and analysis stages are sequential and set the floor on total time.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Running multiple threads concurrently inside one process |
| **Why does it exist?** | Because CPUs gained cores instead of clock speed |
| **Where does it belong?** | Inside a process, above the scheduler |
| **When should I use it?** | CPU-bound, independently splittable work in a language without a GIL |

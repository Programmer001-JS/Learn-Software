# Thread

> **In one line —** one line of execution inside a process, sharing all its memory with every sibling thread — which is what makes threads fast and dangerous at the same time.

| | |
|---|---|
| **Category** | Operating System Concept |
| **Architectural Layer** | User Space |
| **Related notes** | [Process](Process.md) · [Multithreading](Multithreading.md) · [Context Switching](Context%20Switching.md) · [Scheduler](Scheduler.md) · [Event Loop](../05%20-%20Frontend%20Architecture/Event%20Loop.md) |

---

## 1. Short Definition

*What is it?*

A thread is a single sequence of instructions executing inside a [process](Process.md). Every process has at least one. Additional threads share the process's memory, file descriptors and permissions — they have their own [registers](Registers.md) and their own stack, and nothing else of their own.

---

## 2. Purpose

*What is its main purpose?*

To let one program do several things at once: handle multiple requests, keep a user interface responsive while working, or use several CPU cores — without paying the cost of separate processes.

---

## 3. Problem

*What engineering problem does it solve?*

A single-threaded program that calls the database stops completely until the answer arrives. Creating a whole process per concurrent task is far too expensive.

```text
Process creation   ~1–10 ms,  ~50–200 MB
Thread creation    ~10–100 µs, ~1–8 MB stack
```

Threads are the cheap unit of concurrency inside one program.

---

## 4. Architecture Position

```text
┌──────────────── PROCESS ────────────────┐
│  shared: heap, globals, file descriptors│
│                                          │
│  Thread 1        Thread 2      Thread 3 │
│  own stack       own stack     own stack│
│  own registers   own registers own regs │
└──────────────────┬───────────────────────┘
                   ↓
              Scheduler → CPU cores
```

> [!IMPORTANT]
> **What is shared is the whole story.** The heap is shared, so two threads can reach the same object. The stack is private, so local variables are safe. Almost every threading bug lives at that boundary.

---

## 5. Real World Example

- **Java and .NET web servers** use a thread pool: each request is handled by a borrowed thread and returned when done.
- **PostgreSQL deliberately avoids threads** for connections, choosing processes so a crash cannot corrupt the whole server.
- **Browsers** run the JavaScript [event loop](../05%20-%20Frontend%20Architecture/Event%20Loop.md) on one main thread, with Web Workers as separate threads that cannot touch the DOM — a deliberate design to avoid shared-state bugs.
- **Python** has threads, but the **GIL** means only one executes Python bytecode at a time; they help with I/O, never with CPU work.

---

## 6. Input

*What does it receive as input?*

A function to run and its arguments, plus access to everything the process already owns — the heap, globals, open files and sockets.

---

## 7. Processing

*What happens inside it?*

The thread executes its function while the [scheduler](Scheduler.md) grants it CPU time. Its state machine is the same as a process's:

```text
READY  →  RUNNING  →  TERMINATED
             ↕
          BLOCKED  (waiting on I/O or a lock)
```

---

## 8. Output

*What does it return?*

Whatever it computes — but because memory is shared, its most important output is usually a **side effect** on shared state, which is precisely where the danger lies.

---

## 9. Internal Idea

*How does it work internally?*

The kernel treats a thread as its schedulable unit. Each has a stack and a saved register set; the process holds everything else. Creating one is cheap because there is no new address space to build.

> [!CAUTION]
> Because they share the heap, two threads can read and write the same variable at the same time. `counter = counter + 1` is not one operation — it is read, add, write. Two threads interleaving those steps silently lose an increment. That is a **race condition**, and it is invisible until it is not.

---

## 10. Communication

- **Shared memory** — instant, and the source of every race condition
- **Locks / mutexes** — make a section exclusive
- **Queues** — the safest pattern; pass messages instead of sharing state
- **Atomic operations** — lock-free updates for simple counters and flags

---

## 11. Dependencies

- A [process](Process.md) to live inside
- Kernel or runtime threading support (pthreads, Windows threads, or a green-thread runtime)
- Stack memory for each thread — typically 1–8 MB of virtual address space

---

## 12. Alternatives

```text
Thread          real parallelism (outside Python), shared memory, races
    ↓
Process         isolated, expensive, no races by construction
    ↓
Async / event loop   one thread, thousands of I/O tasks, no locks needed
    ↓
Goroutine       user-space scheduling: cheap concurrency plus real parallelism
```

> [!TIP]
> For I/O-bound work, **async is usually the better answer than threads**: no locks, no races, and far less memory per concurrent task.

---

## 13. When To Use

> [!TIP]
> Use threads for CPU-bound parallel work in languages without a GIL (Java, C#, Go, Rust, C++), and for I/O concurrency in languages where async is not available or not idiomatic.

---

## 14. When NOT To Use

> [!CAUTION]
> - **Python CPU-bound work** — the GIL prevents parallel execution; use processes.
> - **Ten thousand concurrent connections** — 10,000 threads means gigabytes of stacks and constant switching. Use an event loop.
> - **When shared mutable state is unavoidable and complex** — the bugs will not be reproducible, and non-reproducible bugs are the worst kind.

---

## 15. Advantages

- Very cheap to create compared with a process
- Instant communication through shared memory
- Real parallelism across cores (in languages without a GIL)
- Keeps interactive applications responsive during background work

---

## 16. Disadvantages

- **Race conditions** — non-deterministic, hard to reproduce, hard to test
- **Deadlocks** — two threads each waiting on the other's lock
- A crash in one thread kills the entire process
- Each thread costs stack memory, so counts do not scale into the thousands
- Locks serialise work, quietly removing the parallelism you added them to protect

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Creation ~10–100 µs; a context switch ~1–10 µs |
| **Memory** | 1–8 MB of stack each — 1,000 threads is gigabytes |
| **CPU** | More threads than cores means switching overhead, not throughput |
| **GPU** | CPU threads typically feed GPU work; the GPU has its own model |
| **Network** | For I/O concurrency, async beats threads at scale |

---

## 18. Security Considerations

> [!CAUTION]
> Threads share **everything** inside the process: memory, credentials, file descriptors. There is no security boundary between them. Untrusted code must never run in a thread of a trusted process — that is what processes and sandboxes are for.

- Race conditions can be exploitable (**TOCTOU** — check a permission, then act on stale information)
- Shared secrets in memory are reachable by every thread
- Thread-local storage separates data by convention, not by enforcement

---

## 19. Mental Model

> [!NOTE]
> **Threads are employees in one office sharing a single filing cabinet.**
>
> They can hand each other documents instantly — far faster than posting a letter between companies. But if two of them edit the same page at the same time, the result is a document that is neither version, and nobody can reconstruct what happened.

---

## 20. Mini Architecture Diagram

```text
┌────────── PROCESS ──────────┐
│  Heap  (SHARED)             │
│  Globals (SHARED)           │
│  File descriptors (SHARED)  │
│                             │
│  T1 stack  T2 stack  T3 stack   (private)
└──────────────┬──────────────┘
               ↓
          Scheduler
               ↓
        Core 0 · Core 1
```

---

## 21. Complete Request Flow

A thread-pool web server:

```text
Server starts → creates a pool of 50 threads
    ↓
Request arrives → borrow an idle thread
    ↓
Thread parses the request
    ↓
Thread calls the database → BLOCKED (releases the CPU, holds its stack)
    ↓
Scheduler runs another thread meanwhile
    ↓
Database responds → thread READY → RUNNING
    ↓
Thread writes the response, returns to the pool
```

> [!IMPORTANT]
> Note the cost: while blocked, the thread still holds megabytes of stack. That is the exact inefficiency the async model removes.

---

## 22. Key Takeaway

> [!IMPORTANT]
> A thread is a cheap unit of execution that shares all of its process's memory — which makes communication instant and correctness genuinely hard.

---

## 23. Common Mistakes

- **Sharing mutable state without a lock** — the classic race condition
- **Locking too broadly**, serialising the work you parallelised
- **Assuming Python threads give CPU parallelism**
- **Creating unbounded threads** instead of using a bounded pool
- **Deadlock through inconsistent lock ordering** — always acquire locks in the same order
- **Testing concurrency only on a fast local machine**, where the race never manifests

---

## 24. Open Source Technologies

- **pthreads** — the POSIX threading standard
- **Java concurrency**, **.NET TPL** — mature thread pool implementations
- **Go goroutines**, **Tokio**, **asyncio** — lighter-weight alternatives
- **ThreadSanitizer**, **Helgrind** — race condition detectors

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Write a counter incremented by two threads without a lock, run it a million times, and observe the lost updates.
- [ ] Find the thread pool size in your web server and work out why that number was chosen.
- [ ] Explain in two sentences why 10,000 threads is a worse answer than an event loop for 10,000 connections.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Process
    ↓
Threads  (shared heap, private stacks)
    ↓
Scheduler
    ↓
CPU cores
```

## 2. Request Flow

```text
Input       a function to execute, plus access to shared process memory
    ↓
Processing  scheduled onto a core; blocks on I/O and locks
    ↓
Output      a result, and side effects on shared state
```

## 3. Real-World Usage

**Java application servers** such as Tomcat handle each request on a thread borrowed from a pool. It works well because the JVM has no GIL — and it is also why tuning that pool size is one of the classic Java performance exercises.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | One line of execution inside a process, sharing its memory |
| **Why does it exist?** | To get concurrency without the cost of separate processes |
| **Where does it belong?** | Inside a process, scheduled onto CPU cores |
| **When should I use it?** | CPU-parallel work without a GIL; prefer async for I/O concurrency |

# Processes and Threads

> **In one line —** a process is a running program with its own memory; a thread is one line of execution inside it, sharing that memory with its siblings.

| | |
|---|---|
| **Category** | Overview note *(hub for this section)* |
| **Architectural Layer** | Operating System |
| **Sub-topics** | [Process](Process.md) · [Thread](Thread.md) · [Multithreading](Multithreading.md) · [Context Switching](Context%20Switching.md) |

---

## 1. The distinction

```text
┌──────────── PROCESS ────────────┐
│  own memory space               │
│  own file descriptors           │
│  isolated from other processes  │
│                                 │
│   ┌────────┐ ┌────────┐         │
│   │Thread 1│ │Thread 2│  ← share all of the above
│   └────────┘ └────────┘         │
│     own stack   own stack       │
└─────────────────────────────────┘
```

| | Process | Thread |
|---|---|---|
| **Memory** | Private | Shared with sibling threads |
| **Creation cost** | High (~ms) | Low (~µs) |
| **Isolation** | Complete | None |
| **Crash impact** | Only itself | Kills the whole process |
| **Communication** | Slow (IPC, sockets, pipes) | Instant (shared variables) |
| **Main risk** | Overhead | Race conditions |

---

## 2. The trade-off in one sentence

> [!IMPORTANT]
> Processes are **safe but expensive**; threads are **cheap but dangerous**. Everything in this section follows from that.

Shared memory is exactly why threads are fast to communicate — and exactly why two threads writing the same variable can corrupt it.

---

## 3. Architecture Position

```text
Operating System
    ↓
Scheduler  ←──────────────┐
    ↓                     │
Process 1   Process 2     │ all threads compete for CPU time
  ├ Thread    ├ Thread ───┘
  └ Thread    └ Thread
    ↓
CPU cores
```

---

## 4. Concurrency vs parallelism

These are constantly confused, and the difference matters.

```text
CONCURRENCY   dealing with many things at once (structure)
              one core, rapidly switching — good for I/O waiting

PARALLELISM   doing many things at once (hardware)
              multiple cores, genuinely simultaneous — good for computation
```

> [!TIP]
> If your work is **waiting** (database, network, disk), you need concurrency — async or threads. If your work is **computing**, you need parallelism — processes or multiple cores. Choosing the wrong one is the most common concurrency mistake there is.

---

## 5. How real systems choose

| System | Model | Why |
|---|---|---|
| **Nginx** | Few processes, event loop | I/O-bound; avoids thread overhead entirely |
| **PostgreSQL** | One process per connection | Isolation — one crashed backend must not take the server down |
| **Gunicorn / Uvicorn** | Multiple worker processes | Python's GIL prevents CPU parallelism within one process |
| **Java / .NET servers** | Thread pools | No GIL; threads give real parallelism |
| **Node.js** | One thread + event loop | I/O-bound by design; parallelism via cluster processes |
| **Go** | Goroutines on a thread pool | User-space scheduling, cheap concurrency plus real parallelism |

---

## 6. Real World Example

- **Chrome** runs each tab as a separate **process**, specifically so one crashing page cannot take the browser down — and so a compromised page cannot read another tab's memory.
- **Celery** workers are separate processes because Python cannot run CPU-bound threads in parallel.
- **A web server handling 10,000 connections** does not use 10,000 threads; each would need megabytes of stack. It uses an event loop instead.

---

## 7. Mental Model

> [!NOTE]
> **A process is a company. Threads are its employees.**
>
> Different companies have separate offices and cannot read each other's files — safe, but talking between them requires a formal channel. Employees within one company share the same office and filing cabinet: instant communication, and two of them editing the same document at once will corrupt it.

---

## 8. Key Takeaway

> [!IMPORTANT]
> Processes buy isolation with overhead; threads buy speed with shared memory and the bugs that come with it.

---

## 9. Common Mistakes

- **Confusing concurrency with parallelism**, and applying the wrong one
- **Adding threads to an I/O-bound system** — they will all wait, just in parallel
- **Assuming Python threads give CPU parallelism** — the GIL prevents it
- **Sharing mutable state between threads without a lock**
- **Creating unbounded threads or processes** — always use a pool with a limit

---

## 10. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 11. Workbook Exercise

- [ ] Determine whether your application is I/O-bound or CPU-bound, with a measurement rather than a guess.
- [ ] Find out how many worker processes and threads your server actually runs, and why that number was chosen.
- [ ] Explain concurrency vs parallelism in two sentences without using the words "concurrency" or "parallelism".

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Operating System
    ↓
Scheduler
    ↓
Processes  →  Threads
    ↓
CPU cores
```

## 2. Request Flow

```text
Input       work that must be done
    ↓
Processing  distributed across processes (isolation) or threads (shared memory)
    ↓
Output      results, plus whatever contention the chosen model created
```

## 3. Real-World Usage

**Chrome's** multi-process architecture was a deliberate trade: far higher memory usage in exchange for a browser where one bad page cannot crash or spy on the rest. That is the process/thread trade-off made visible in a consumer product.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The two units of execution an operating system provides |
| **Why does it exist?** | To run many things at once on limited hardware |
| **Where does it belong?** | Between the scheduler and the CPU |
| **When should I use it?** | Processes for isolation and CPU work; threads or async for I/O waiting |

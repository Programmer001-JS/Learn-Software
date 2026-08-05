# Context Switching

> **In one line —** saving everything one thread was doing and restoring everything another was doing, so that a few cores can pretend to be hundreds.

| | |
|---|---|
| **Category** | Operating System Mechanism |
| **Architectural Layer** | Kernel Space |
| **Cost** | ~1–10 µs, plus lost cache warmth |
| **Related notes** | [Scheduler](Scheduler.md) · [Thread](Thread.md) · [Process](Process.md) · [Registers](Registers.md) · [Cache Memory](Cache%20Memory.md) |

---

## 1. Short Definition

*What is it?*

A context switch is the act of taking one [thread](Thread.md) off a CPU core and putting another one on. The kernel saves the outgoing thread's complete state and restores the incoming thread's, so each resumes exactly where it left off.

---

## 2. Purpose

*What is its main purpose?*

To make multitasking possible. Without it, a core could only ever run one thread to completion, and a machine with 8 cores could run 8 programs.

---

## 3. Problem

*What engineering problem does it solve?*

Hundreds of threads want CPU time; there are a handful of cores. Switching between them fast enough makes every thread appear to run continuously.

---

## 4. Architecture Position

```text
Threads waiting  [T1] [T2] [T3] [T4]
                        ↓
                   SCHEDULER decides
                        ↓
                  CONTEXT SWITCH        ← you are here
                        ↓
                    CPU core
```

---

## 5. Real World Example

- **Your machine running 300 processes on 8 cores** — thousands of switches per second, invisible to you.
- **Kubernetes CPU throttling** — a container that exceeds its quota is descheduled, producing latency spikes that look like application slowness but are pure scheduling.
- **Go, Node.js and Python async** exist largely to *avoid* this cost: they multiplex thousands of tasks onto a few OS threads, switching in user space where it costs nanoseconds instead of microseconds.

---

## 6. Input

*What does it receive as input?*

A decision from the [scheduler](Scheduler.md), triggered by one of:

- The **timer interrupt** — the thread's time slice expired
- The thread **blocked** on I/O or a lock
- A **higher-priority thread** became runnable
- The thread **finished** or yielded voluntarily

---

## 7. Processing

*What happens inside it?*

```text
1. Interrupt or syscall → enter kernel mode
    ↓
2. SAVE the current thread's state:
       all registers, program counter, stack pointer, flags
    ↓
3. Update its accounting (CPU time used)
    ↓
4. Scheduler picks the next thread
    ↓
5. LOAD that thread's saved state
    ↓
6. If it belongs to a DIFFERENT PROCESS:
       switch page tables → flush the TLB      ← the expensive part
    ↓
7. Return to user mode; the new thread resumes
```

---

## 8. Output

*What does it return?*

A different thread running on the core, resuming as if it had never stopped — and a small, unavoidable amount of wasted time.

---

## 9. Internal Idea

*How does it work internally?*

The direct cost is copying [registers](Registers.md): roughly 1–10 microseconds. The **indirect** cost is usually larger and invisible in profilers:

> [!IMPORTANT]
> The new thread's data is not in the [CPU cache](Cache%20Memory.md). It runs slowly for a while as the cache refills — this is called **cache pollution**, and it often costs more than the switch itself. Switching between threads of different processes also flushes the TLB, adding page-table lookups on every memory access until it warms up again.

---

## 10. Communication

- **[Scheduler](Scheduler.md)** — decides when it happens
- **[Registers](Registers.md)** and the process control block — what gets saved and restored
- **MMU / TLB** — flushed on a process switch
- **[Cache](Cache%20Memory.md)** — silently degraded by every switch

---

## 11. Dependencies

- Hardware **timer interrupts**, so the kernel can regain control
- Per-thread saved-state structures
- MMU support for switching address spaces

---

## 12. Alternatives

```text
OS context switch      ~1–10 µs      kernel-managed, general
    ↓
User-space switching   ~10–100 ns    goroutines, async/await, fibers
    ↓
Event loop             no switching at all — one thread, many I/O tasks
    ↓
CPU pinning            avoid migration between cores, keeping caches warm
```

> [!TIP]
> This cost is the entire reason async programming exists. An event loop handles 10,000 connections with zero context switches, where 10,000 threads would spend most of the CPU switching between them.

---

## 13. When To Use

> [!TIP]
> It is not something you use — it is something you **budget for**. Think about it when choosing how many threads or workers to run.

---

## 14. When NOT To Use

> [!CAUTION]
> Avoid designs that cause excessive switching:
> - Far more runnable threads than cores
> - Very short time slices with heavy lock contention
> - Threads that constantly block and wake on fine-grained locks
>
> The symptom is high CPU usage with low throughput — the machine is busy switching rather than working. Watch `vmstat`'s `cs` column.

---

## 15. Advantages

- Makes multitasking and preemption possible at all
- Prevents any single thread from monopolising a core
- Lets blocked threads release the CPU immediately
- Completely transparent to application code

---

## 16. Disadvantages

- Direct cost of ~1–10 µs per switch
- Cache and TLB pollution, often the larger hidden cost
- Process switches cost considerably more than thread switches
- At high thread counts, becomes the dominant use of CPU time

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | 1–10 µs direct, plus a slow period while caches refill |
| **Memory** | Each thread's saved state and stack |
| **CPU** | Pure overhead — no useful work happens during a switch |
| **GPU** | GPU context switches are far more expensive; this is why batching matters |
| **Network** | I/O-bound threads switch constantly, which is exactly why async wins there |

---

## 18. Security Considerations

> [!CAUTION]
> A context switch writes all [registers](Registers.md) — including any secrets currently held in them — into kernel memory. Those values can appear in crash dumps.

- **Timing side channels** — switch timing has been used to infer another process's behaviour on shared hardware
- **Spectre mitigations** made switches measurably more expensive, because caches and prediction state must be flushed more aggressively

---

## 19. Mental Model

> [!NOTE]
> **A context switch is stopping work on one document to start another.**
>
> You must note exactly where you were, put every page away, take out the other document, and remember where *that* one was. The paperwork is quick. Getting your head back into the second document — that is the cache warming up, and it is the part that really costs you.

---

## 20. Mini Architecture Diagram

```text
Thread A RUNNING
    ↓
Interrupt / block / preemption
    ↓
Save A: registers, PC, stack pointer
    ↓
Scheduler picks B
    ↓
Load B: registers, PC, stack pointer
(+ page tables and TLB flush if a different process)
    ↓
Thread B RUNNING  — caches cold for a while
```

---

## 21. Complete Request Flow

One request in a thread-per-request server:

```text
Thread handles the request → RUNNING
    ↓
Calls the database → BLOCKED
    ↓
CONTEXT SWITCH #1 — another thread runs
    ↓
Database responds (~5 ms later) → thread READY
    ↓
CONTEXT SWITCH #2 — back to our thread, caches now cold
    ↓
Builds the response
    ↓
Writes to the socket → possibly BLOCKED again
    ↓
CONTEXT SWITCH #3
```

Three switches for one request. At 10,000 requests per second that is 30,000 switches — and the async model reduces it to nearly zero.

---

## 22. Key Takeaway

> [!IMPORTANT]
> A context switch costs microseconds directly and much more in cold caches — which is why more threads is not more speed, and why async I/O exists.

---

## 23. Common Mistakes

- **Creating far more threads than cores** and expecting throughput to improve
- **Ignoring the cache cost**, which does not appear as its own line in a profiler
- **Blaming the application** for latency that is really scheduling and throttling
- **Not monitoring the context switch rate** — `vmstat`'s `cs` column is an early warning of thrashing
- **Using threads for I/O** where an event loop would eliminate the switching entirely

---

## 24. Open Source Technologies

- **vmstat**, **pidstat**, **perf sched** — measure switch rates and latency
- **Go runtime**, **Tokio**, **asyncio**, **Java virtual threads** — user-space alternatives
- **taskset**, **cgroups cpuset** — pin threads to cores to keep caches warm
- **io_uring** — reduces both syscalls and switches for I/O-heavy servers

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Run `vmstat 1` under load and watch the `cs` column. What happens when you double your worker count?
- [ ] Compare a thread-per-request server with an async one at 1,000 concurrent connections.
- [ ] Explain in two sentences why the cache cost of a switch is usually worse than the switch itself.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Scheduler
    ↓
Context switch  (save → select → restore)
    ↓
CPU core
```

## 2. Request Flow

```text
Input       a scheduling decision
    ↓
Processing  save registers → pick next thread → restore registers → flush TLB if needed
    ↓
Output      a different thread running, with cold caches
```

## 3. Real-World Usage

**Go's runtime** multiplexes hundreds of thousands of goroutines onto a small pool of OS threads, switching between them in user space. Avoiding kernel context switches is the single largest reason Go handles massive concurrency on modest hardware.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Saving one thread's state and restoring another's on a CPU core |
| **Why does it exist?** | Because there are far more threads than cores |
| **Where does it belong?** | Inside the kernel, executed on behalf of the scheduler |
| **When should I use it?** | Not a choice — a cost to budget for when sizing concurrency |

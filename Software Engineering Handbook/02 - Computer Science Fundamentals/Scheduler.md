# Scheduler

> **In one line —** the part of the kernel that decides which process gets the CPU next, and for how long, thousands of times per second.

| | |
|---|---|
| **Category** | Kernel Subsystem |
| **Architectural Layer** | Kernel Space |
| **Related notes** | [Kernel](Kernel.md) · [Process](Process.md) · [Thread](Thread.md) · [Context Switching](Context%20Switching.md) · [CPU](CPU.md) |

---

## 1. Short Definition

*What is it?*

The scheduler is the kernel component that shares the [CPU](CPU.md) between all runnable [processes](Process.md) and [threads](Thread.md). It repeatedly answers one question: *who runs next, and for how long?*

---

## 2. Purpose

*What is its main purpose?*

To create the illusion that every program has its own processor, on a machine that has only a handful of cores — and to do it fairly, without letting any one program starve the others.

---

## 3. Problem

*What engineering problem does it solve?*

A laptop runs hundreds of processes on perhaps eight cores. Without a scheduler, the first program to start would run until it finished and nothing else would ever respond.

```text
8 cores, 300 processes
    ↓
Scheduler gives each a few milliseconds in turn
    ↓
Everything appears to run simultaneously
```

---

## 4. Architecture Position

```text
Processes and threads  (user space)
    ↓  all want CPU time
┌──── KERNEL ────────────────────────┐
│  SCHEDULER                         │
│    ready queue → pick next → run   │
└────────────────┬───────────────────┘
                 ↓
             CPU cores
```

---

## 5. Real World Example

- **Your machine feeling responsive** while compiling, playing music and running a browser is entirely the scheduler's work.
- **Kubernetes CPU limits** are enforced through the Linux CFS scheduler's bandwidth control — this is why a container that hits its CPU limit is *throttled* rather than killed.
- **Database latency spikes** under load are often scheduling delays, not query slowness: the query was ready, the CPU was busy.

---

## 6. Input

*What does it receive as input?*

- The **ready queue** — every thread that could run right now
- Each thread's **priority** or weight (`nice` value, real-time class)
- **Events** — a timer tick, a thread blocking on I/O, a thread becoming runnable

---

## 7. Processing

*What happens inside it?*

```text
Timer interrupt fires (or a thread blocks/wakes)
    ↓
Scheduler runs
    ↓
Pick the highest-priority runnable thread
    ↓
CONTEXT SWITCH — save the old thread's registers, load the new one's
    ↓
New thread runs for its time slice
```

Linux's default scheduler (**CFS** — Completely Fair Scheduler) tracks how much CPU time each thread has already received and always picks the one that has had the least, weighted by priority.

---

## 8. Output

*What does it return?*

A decision, executed as a [context switch](Context%20Switching.md). The visible effect is that everything appears to run at once.

---

## 9. Internal Idea

*How does it work internally?*

Two states matter more than anything else:

```text
RUNNING     currently on a core
READY       could run, waiting for a core
BLOCKED     waiting for I/O — NOT the scheduler's problem
```

> [!IMPORTANT]
> A thread blocked on a database query or a network read is **not** consuming CPU. This is why a server can be at 5% CPU and still be completely saturated: the bottleneck is waiting, not computing. Adding CPU cores to an I/O-bound system changes nothing.

---

## 10. Communication

- **All processes and threads** — indirectly, by being scheduled
- **[CPU](CPU.md) cores** — assigning work to each
- **The timer** — the interrupt that gives the scheduler its regular chance to intervene
- **cgroups** — the mechanism containers use to enforce CPU shares and limits

---

## 11. Dependencies

- A hardware **timer interrupt**, so the kernel can regain control from a running program
- Support for [context switching](Context%20Switching.md)
- Per-thread accounting structures

---

## 12. Alternatives

```text
Preemptive       kernel forcibly interrupts    ← every modern OS
Cooperative      threads must yield voluntarily  (old Windows, some embedded)
Real-time        strict deadlines, guaranteed latency  (avionics, audio)
User-space       goroutines, async/await — scheduling inside one OS thread
```

> [!TIP]
> Go's goroutines and Python's `asyncio` are **user-space schedulers**. They avoid kernel context switches entirely, which is why they handle hundreds of thousands of concurrent tasks that OS threads could not.

---

## 13. When To Use

> [!TIP]
> You never invoke the scheduler, but you should think about it when tuning concurrency: how many worker processes, how many threads, and what CPU limits to set on a container.

---

## 14. When NOT To Use

> [!CAUTION]
> Do not try to outsmart the scheduler with thread priorities or CPU pinning in ordinary applications. It almost always makes things worse, and the actual problem is usually I/O waiting or lock contention.

---

## 15. Advantages

- Hundreds of processes share a few cores smoothly
- No program can monopolise the machine
- Priorities allow important work to be favoured
- Automatic, requiring nothing from application code

---

## 16. Disadvantages

- Every switch costs time and cache locality
- Too many runnable threads produces thrashing — more switching than working
- Fairness is not the same as low latency; a latency-sensitive task can still wait
- Behaviour differs between operating systems, so tuning is not portable

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Each context switch costs ~1–10 µs plus lost cache warmth |
| **Memory** | Each thread needs its own stack — typically 1–8 MB |
| **CPU** | Excess threads mean the CPU spends its time switching rather than working |
| **GPU** | The GPU has its own scheduling; the CPU one only feeds it |
| **Network** | I/O-bound threads sit blocked, consuming no CPU at all |

> [!IMPORTANT]
> More threads is not more speed. Beyond roughly the number of cores for CPU-bound work, additional threads only add switching overhead.

---

## 18. Security Considerations

> [!CAUTION]
> Scheduling is a **side channel**. By measuring how long its own work takes, one process can infer what another is doing — the basis of several cross-tenant attacks on shared cloud hardware.

- A process that never yields can degrade the whole machine, so cgroup CPU limits are a real defence
- Timing-based information leaks are why cryptographic code must be constant-time

---

## 19. Mental Model

> [!NOTE]
> **The scheduler is a teacher with thirty students and one microphone.**
>
> Everyone gets a short turn, so everyone feels heard. Handing the microphone over takes a moment — do it too often and the lesson is nothing but handovers.

---

## 20. Mini Architecture Diagram

```text
Ready queue:   [T1] [T2] [T3] [T4]
                    ↓
                SCHEDULER
                    ↓
        ┌───────┬───────┬───────┐
      Core 0  Core 1  Core 2  Core 3
                    ↓
        blocked threads wait elsewhere (on I/O)
```

---

## 21. Complete Request Flow

One thread's life during an HTTP request:

```text
Thread READY  → scheduler picks it → RUNNING
    ↓
Parses the request  (CPU work)
    ↓
Calls the database → BLOCKED     ← releases the CPU immediately
    ↓
Scheduler runs a different thread
    ↓
Database responds → thread becomes READY
    ↓
Scheduler picks it again → RUNNING
    ↓
Builds the response, writes to the socket → done
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> The scheduler shares a few cores among many threads — and a thread waiting on I/O costs no CPU at all, which is why "slow" and "CPU-busy" are entirely different problems.

---

## 23. Common Mistakes

- **Adding threads to fix an I/O-bound system** — they will all wait, just in parallel
- **Assuming high CPU means the bottleneck is the CPU** — check whether it is your process or the kernel
- **Setting container CPU limits too low**, causing invisible throttling and latency spikes
- **Confusing concurrency with parallelism** — concurrency is structure, parallelism is hardware
- **Fiddling with thread priorities** instead of finding the real contention

---

## 24. Open Source Technologies

- **Linux CFS** and **EEVDF** — the mainline schedulers
- **cgroups v2** — CPU limits and shares, used by Docker and Kubernetes
- **Go runtime**, **Tokio**, **asyncio** — user-space schedulers
- **htop**, **pidstat**, **perf sched** — observe scheduling behaviour

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Load-test your application and record whether it is CPU-bound or I/O-bound. Justify the answer with a measurement.
- [ ] Check the CPU limits on your containers and whether they are being throttled.
- [ ] Explain in two sentences why 200 threads on 8 cores can be slower than 16 threads.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Threads and processes
    ↓
Scheduler (kernel)
    ↓
CPU cores
```

## 2. Request Flow

```text
Input       the set of runnable threads plus their priorities
    ↓
Processing  pick the most deserving, context switch to it
    ↓
Output      one thread running per core, everything appearing simultaneous
```

## 3. Real-World Usage

**Kubernetes** enforces CPU requests and limits through the Linux scheduler's cgroup bandwidth control. A pod exceeding its limit is throttled by the scheduler — which is why mysterious latency in Kubernetes so often turns out to be a CPU limit set too tightly.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The kernel component deciding which thread runs next |
| **Why does it exist?** | Because there are far more threads than cores |
| **Where does it belong?** | Inside the kernel, between threads and CPU cores |
| **When should I use it?** | Think about it when tuning concurrency, worker counts and CPU limits |

# Runtime Explained

> **In one line —** everything that must exist *while your program runs* but that you did not write: memory management, the standard library, the scheduler, the garbage collector.

| | |
|---|---|
| **Category** | Foundation Concept |
| **Architectural Layer** | Between application and operating system |
| **Related notes** | [Virtual Machines](Virtual%20Machines.md) · [Bytecode](Bytecode.md) · [CPython](CPython.md) · [JVM](JVM.md) · [Node.js Runtime](Node.js%20Runtime.md) · [Backend Fundamentals](../06%20-%20Backend%20Architecture/Backend%20Fundamentals.md) |

---

## 1. Short Definition

*What is it?*

A runtime is the software environment that supports a program **during execution**. It handles memory, provides the standard library, manages threads or an event loop, and translates your code's needs into [system calls](../02%20-%20Computer%20Science%20Fundamentals/System%20Calls.md).

---

## 2. Purpose

*What is its main purpose?*

To let you write `open("file.txt")` instead of managing file descriptors, and `x = [1,2,3]` instead of calculating memory offsets. The runtime absorbs the machinery so your code can be about the problem.

---

## 3. Problem

*What engineering problem does it solve?*

Without a runtime, every program would have to implement memory allocation, string handling, I/O buffering, threading and error propagation itself. That work is identical for every program, so it was factored out.

---

## 4. Architecture Position

```text
Your application code
    ↓
Framework  (FastAPI, Express, Spring)
    ↓
Standard library
    ↓
RUNTIME  (CPython, Node.js, JVM, .NET, Go runtime)   ← you are here
    ↓
Operating system  (system calls)
    ↓
Hardware
```

> [!IMPORTANT]
> This is one of the most useful layer diagrams in the handbook. When something behaves strangely — memory growth, latency spikes, blocked requests — knowing which of these layers owns the behaviour is most of the diagnosis.

---

## 5. What a runtime actually provides

| Responsibility | Example |
|---|---|
| **Memory management** | Allocation, garbage collection |
| **Standard library** | Strings, collections, dates, JSON, HTTP |
| **Concurrency** | Threads, an event loop, goroutines |
| **I/O** | Buffering, async, socket handling |
| **Error handling** | Exceptions, stack traces, panics |
| **Startup** | Loading modules, initialising the heap |
| **Interop** | Calling native C libraries |

---

## 6. Real World Example

- **[Node.js](Node.js%20Runtime.md)** — V8 plus libuv; the event loop that makes JavaScript viable on a server is a runtime feature, not a language feature.
- **[CPython](CPython.md)** — reference counting, the GIL and the bytecode interpreter are all runtime properties, and they shape how Python applications must be deployed.
- **Go runtime** — compiled binaries still embed a runtime that provides goroutine scheduling and garbage collection.
- **[JVM](JVM.md)** — its JIT and garbage collectors are the reason Java performs as well as it does.

---

## 7. Input and Output

*What does it receive, and what does it produce?*

**Input:** your compiled code or bytecode, plus configuration — heap size, worker counts, GC settings, environment variables.

**Output:** a running process that behaves according to your code, with all the resource management done underneath.

---

## 8. Internal Idea

*How does it work internally?*

A runtime is essentially a **long-running service inside your process**. It starts before your code, sets up the heap and the scheduler or event loop, runs your program, and keeps working in the background — collecting garbage, waking coroutines, buffering I/O — until the process exits.

---

## 9. Garbage collection — the runtime feature you will actually notice

```text
You allocate objects
    ↓
Runtime tracks which are still reachable
    ↓
Periodically frees the unreachable ones
    ↓
Sometimes PAUSES your program to do it       ← the part that shows up in latency graphs
```

> [!TIP]
> GC pauses are a common and often unrecognised cause of p99 latency spikes. If your average response time is fine but the worst 1% is terrible, garbage collection is one of the first things to check.

---

## 10. Communication

- **Your code** above it
- **The [operating system](../02%20-%20Operating%20Systems.md)** below it, via [system calls](../02%20-%20Computer%20Science%20Fundamentals/System%20Calls.md)
- **Native libraries** through a foreign function interface
- **Its own background threads** — GC, JIT compilation, timers

---

## 11. Dependencies

- An operating system providing processes, memory and I/O
- Often a specific version — Python 3.11, Node 20, JDK 21. **The runtime version is part of your deployment**, and mismatches are a frequent cause of "works on my machine".

---

## 12. Comparison of common runtimes

| Runtime | Concurrency model | Memory | Startup | Best at |
|---|---|---|---|---|
| **CPython** | Threads limited by the GIL; asyncio | Reference counting + cycle GC | ~50 ms | Scripting, data, AI |
| **Node.js** | Single-threaded event loop | V8 generational GC | ~50 ms | I/O-heavy APIs |
| **JVM** | Real threads, virtual threads | Sophisticated GCs | ~1–3 s | Long-running, high-throughput services |
| **.NET** | Real threads, async/await | Generational GC | ~100 ms | Enterprise services |
| **Go** | Goroutines on a thread pool | Low-pause GC | ~1 ms | Network services, CLIs |

---

## 13. When To Use

> [!TIP]
> You choose a runtime whenever you choose a language, so choose deliberately: match the runtime's concurrency model to your workload. I/O-heavy work suits event loops; CPU-heavy work needs real parallelism.

---

## 14. When NOT To Use

> [!CAUTION]
> Fighting your runtime's model rarely ends well. CPU-bound Python threads will not use multiple cores. Blocking calls inside a Node.js event loop stall every other request. Choose a different runtime rather than working around the one you have.

---

## 15. Advantages and Disadvantages

**Advantages**
- Removes enormous amounts of repetitive, error-prone work
- Memory safety and automatic cleanup
- Cross-platform behaviour
- Mature libraries and tooling

**Disadvantages**
- Memory and startup overhead
- GC pauses you cannot fully control
- Must be shipped or installed alongside the application
- Its model constrains your architecture, whether you like it or not

---

## 16. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | JIT gets close to native; interpretation costs 10–100× |
| **Memory** | Baseline footprint before your code allocates anything |
| **CPU** | GC and JIT consume real CPU in the background |
| **Startup** | ~1 ms (Go) to several seconds (JVM) — decisive for serverless |
| **Container size** | The runtime dominates image size for interpreted languages |

---

## 17. Security Considerations

> [!CAUTION]
> **The runtime version is a security dependency.** Outdated Node, Python or JVM versions carry known, published vulnerabilities, and "we never upgraded the runtime" is a common root cause in incident reports.

- Runtimes with dynamic evaluation (`eval`, `pickle`, deserialisation) offer direct remote-code-execution paths if fed user input
- Dependencies run with the full privileges of your process — the supply chain is part of your attack surface
- Stack traces leak internal paths and library versions; do not return them to users

---

## 18. Mental Model

> [!NOTE]
> **The runtime is the stage crew of a theatre production.**
>
> The audience watches the actors (your code). Behind them, the crew manages lighting, props and scene changes. When they do their job, nobody notices; when the garbage collector pauses mid-scene, everybody does.

---

## 19. Mini Architecture Diagram

```text
Application
    ↓
Framework
    ↓
Standard library
    ↓
RUNTIME
  ├─ memory manager / GC
  ├─ scheduler / event loop
  ├─ JIT compiler
  └─ I/O subsystem
    ↓
System calls
    ↓
Kernel → Hardware
```

---

## 20. Complete Request Flow

An HTTP request through a Python API:

```text
Packet arrives → kernel → socket
    ↓
Runtime's event loop (asyncio) wakes the waiting coroutine
    ↓
Framework routes it to your handler
    ↓
Your code runs; the runtime allocates objects on the heap
    ↓
Database call → runtime issues a syscall, suspends the coroutine
    ↓
Other requests proceed meanwhile
    ↓
Response serialised, written to the socket
    ↓
Objects become unreachable → GC reclaims them later
```

---

## 21. Key Takeaway

> [!IMPORTANT]
> The runtime is the invisible layer between your code and the OS — and its concurrency model, garbage collector and startup cost shape your architecture more than the language syntax ever will.

---

## 22. Common Mistakes

- **Ignoring the runtime version** in deployment, then debugging "works on my machine"
- **Blocking an event loop** with synchronous work
- **Expecting CPU parallelism from Python threads**
- **Never tuning heap or GC settings** on the JVM and blaming latency on the application
- **Underestimating startup time** in serverless environments
- **Treating the runtime as free** — it costs memory and image size before your code does anything

---

## 23. Open Source Technologies

- **CPython**, **PyPy** — Python runtimes
- **Node.js**, **Deno**, **Bun** — JavaScript runtimes
- **OpenJDK**, **GraalVM** — JVM implementations
- **.NET runtime**, **Go runtime**
- **py-spy**, **async-profiler**, **clinic.js** — see what the runtime is actually doing

---

## 24. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 25. Workbook Exercise

- [ ] Find the exact runtime version your production application uses, and check whether it is still supported.
- [ ] Measure your application's baseline memory before handling any traffic.
- [ ] Explain in two sentences how your runtime handles concurrency, and whether it matches your workload.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Application
    ↓
Framework
    ↓
Runtime
    ↓
Operating system
    ↓
Hardware
```

## 2. Request Flow

```text
Input       your code plus runtime configuration
    ↓
Processing  memory management, scheduling, I/O, JIT — all beneath your code
    ↓
Output      a running process, with resources managed for you
```

## 3. Real-World Usage

**Node.js** made JavaScript a serious server language not by changing the language but by pairing V8 with libuv's event loop. The runtime, not the syntax, is what made it good at handling thousands of concurrent connections.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The environment that supports your program while it runs |
| **Why does it exist?** | So every program does not reimplement memory, I/O and concurrency |
| **Where does it belong?** | Between your code and the operating system |
| **When should I use it?** | Always — the real decision is which one, and why |

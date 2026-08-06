# .NET Runtime

> **In one line —** Microsoft's answer to the JVM: bytecode, a JIT compiler and a garbage collector — now open source and cross-platform, with first-class async built into the language.

| | |
|---|---|
| **Also called** | CLR — Common Language Runtime |
| **Category** | Language Runtime |
| **Architectural Layer** | Runtime |
| **Languages** | C#, F#, VB.NET |
| **Related notes** | [Bytecode](Bytecode.md) · [JVM](JVM.md) · [Virtual Machines](Virtual%20Machines.md) · [ASP.NET Core](../06%20-%20Backend%20Architecture/ASP.NET%20Core.md) · [Entity Framework](../08%20-%20Databases%20and%20Data/Entity%20Framework.md) |

---

## 1. Short Definition

*What is it?*

The .NET runtime executes **CIL** (Common Intermediate Language) [bytecode](Bytecode.md) produced by C#, F# and VB.NET. It JIT-compiles that bytecode to machine code, manages memory with a generational garbage collector, and provides the standard library.

---

## 2. Purpose

*What is its main purpose?*

To run multiple languages on one runtime, on any operating system, at close to native speed, with memory safety and an unusually well-integrated async model.

---

## 3. Problem

*What engineering problem does it solve?*

The same problems the [JVM](JVM.md) addressed — portability and memory safety — plus one more: allowing several languages to share one runtime, one type system and one standard library.

Historically .NET was Windows-only. **.NET Core** (2016) rebuilt it as open source and cross-platform, which is why .NET is now a legitimate choice for Linux containers.

---

## 4. Architecture Position

```text
C# / F# source
    ↓  Roslyn compiler
CIL bytecode (.dll)
    ↓
┌────────── .NET RUNTIME (CLR) ──────────┐
│  Assembly loader                       │
│  JIT compiler (RyuJIT)                 │
│  Generational garbage collector        │
│  Thread pool + async machinery         │
└──────────────────┬─────────────────────┘
                   ↓
        Operating system → hardware
```

---

## 5. Real World Example

- **Stack Overflow** famously served enormous traffic from a small number of .NET servers — a well-known demonstration of how far a well-tuned monolith can go.
- **Microsoft's own cloud services**, Bing and much of Azure's control plane.
- **Enterprise line-of-business systems** — the largest single segment of .NET usage.
- **Unity game engine** — uses C# and a .NET-derived runtime for game scripting.

---

## 6. Input and Processing

**Input:** CIL bytecode in `.dll` assemblies, plus runtime configuration.

**Processing:**

```text
Assembly loaded and verified
    ↓
Method called for the first time → JIT-compiled to machine code
    ↓
Compiled code cached for subsequent calls
    ↓
Tiered compilation: quick first, re-optimised if it stays hot
    ↓
GC reclaims unreachable objects generationally
```

> [!IMPORTANT]
> Unlike the JVM, .NET JIT-compiles **on first call** rather than after a threshold. Startup is therefore faster than the JVM's, and peak throughput arrives sooner.

---

## 7. Output

A running application with JIT-compiled machine code and automatic memory management.

---

## 8. Internal Idea — async/await

.NET's most distinctive feature is that asynchronous programming is built into the language and compiler, not bolted on.

```text
await SomeIoOperation()
    ↓
Compiler rewrites the method into a state machine
    ↓
The thread is RELEASED while waiting — it is not blocked
    ↓
When the I/O completes, execution resumes on a thread pool thread
```

> [!TIP]
> This is why ASP.NET Core handles very high concurrency with a small thread pool: waiting costs a state machine object rather than a whole blocked thread.

---

## 9. Communication

- **Operating system** — through system calls; the same runtime works on Windows, Linux and macOS
- **Native libraries** — via P/Invoke
- **Other services** — HTTP, gRPC, message queues
- **Diagnostics** — EventPipe, dotnet-counters, dotnet-trace

---

## 10. Dependencies

- The .NET runtime installed, or a self-contained publish that bundles it
- A matching runtime major version
- For AOT builds, a supported platform and no runtime code generation

---

## 11. Alternatives

```text
.NET (JIT)          fast startup, high throughput, cross-platform
    ↓
.NET Native AOT     compile ahead of time: millisecond startup, small binary, some feature limits
    ↓
JVM                 very similar model; larger ecosystem in data infrastructure
    ↓
Go                  simpler runtime, smaller binaries, less sophisticated JIT
    ↓
Node.js             single-threaded event loop; better for pure I/O fan-out, worse for CPU work
```

---

## 12. When To Use

> [!TIP]
> Use .NET for long-running services that need real threading, strong typing and high throughput — particularly where the team already knows C# or the ecosystem (Entity Framework, Azure integration) fits.

---

## 13. When NOT To Use

> [!CAUTION]
> - **Very small serverless functions** — use Native AOT, or a lighter runtime.
> - **Where the surrounding ecosystem is Python or JavaScript** — data science and AI tooling are far weaker on .NET.
> - **When the team has no C# experience** and there is no other reason to choose it.

---

## 14. Advantages

- Fast startup relative to the JVM, with strong peak throughput
- Real multithreading — no GIL
- `async`/`await` designed into the language, not retrofitted
- Excellent tooling: Visual Studio, Rider, built-in diagnostics
- Cross-platform and fully open source since .NET Core
- Native AOT for millisecond startup when needed

---

## 15. Disadvantages

- Smaller open-source ecosystem than the JVM or Python in several domains
- Historical Windows association still shapes hiring and perception
- GC pauses affect tail latency, as with any managed runtime
- Runtime and framework version churn has been rapid

---

## 16. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Near-native; among the fastest managed runtimes |
| **Memory** | Lower baseline than the JVM, higher than Go |
| **CPU** | Real parallelism across cores; JIT and GC run in the background |
| **Startup** | ~100 ms typical; ~10 ms with Native AOT |
| **Container** | ~100–200 MB; far smaller with AOT and trimming |

---

## 17. Security Considerations

> [!CAUTION]
> **`BinaryFormatter` deserialisation** was a serious remote-code-execution vector and is now obsolete and disabled by default. Deserialising untrusted data remains dangerous in every managed runtime.

- Keep the runtime patched; Microsoft ships regular security releases
- CIL decompiles very cleanly — ILSpy reconstructs near-original C#
- Dependencies from NuGet run with your process's full privileges
- ASP.NET Core has good security defaults, but configuration mistakes still dominate real incidents

---

## 18. Mental Model

> [!NOTE]
> **.NET is the JVM's sibling raised in a different house.**
>
> Same architecture — bytecode, JIT, garbage collector — different ecosystem, different tooling culture, and a noticeably better answer to asynchronous I/O built into the language itself.

---

## 19. Mini Architecture Diagram

```text
C# source
    ↓ Roslyn
CIL bytecode (.dll)
    ↓
CLR: loader → JIT (RyuJIT) → machine code
    ↓
Managed heap ←── generational GC
    ↓
Thread pool / async state machines
    ↓
Operating system → hardware
```

---

## 20. Complete Request Flow

An ASP.NET Core request:

```text
Runtime starts, assemblies loaded  (~100 ms, once)
    ↓
Request arrives on a Kestrel thread
    ↓
Middleware pipeline runs
    ↓
Controller action invoked (JIT-compiled on first call)
    ↓
await database query → THREAD RELEASED back to the pool
    ↓
Other requests use that thread meanwhile
    ↓
Query completes → continuation scheduled on a pool thread
    ↓
Response serialised and written
    ↓
Objects become garbage; GC reclaims them generationally
```

---

## 21. Key Takeaway

> [!IMPORTANT]
> .NET is a JIT-compiled, garbage-collected, cross-platform runtime with the best-integrated async model of the major managed platforms — strong for long-running, concurrent services.

---

## 22. Common Mistakes

- **Blocking on async code** (`.Result`, `.Wait()`) — a classic cause of thread-pool starvation and deadlocks
- **Confusing .NET Framework with modern .NET** — the old Windows-only Framework is a different product
- **Not setting container memory limits**, so the GC misjudges available memory
- **Using `BinaryFormatter`** or deserialising untrusted input
- **Ignoring `ConfigureAwait`** in library code
- **Assuming AOT is a drop-in** — reflection-heavy libraries often will not work

---

## 23. Open Source Technologies

- **.NET runtime and SDK** — open source, cross-platform
- **ASP.NET Core** — the web framework
- **Entity Framework Core** — the standard ORM
- **Roslyn** — the compiler platform, usable as a library
- **dotnet-counters**, **dotnet-trace**, **PerfView** — diagnostics

---

## 24. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 25. Workbook Exercise

- [ ] Write the same endpoint with and without `async`, then load-test both and compare thread usage.
- [ ] Publish an application as Native AOT and compare startup time and image size.
- [ ] Explain in two sentences why `.Result` on an async call can deadlock a web request.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
C# source
    ↓
CIL bytecode
    ↓
.NET runtime (JIT + GC)
    ↓
Machine code
    ↓
Operating system → hardware
```

## 2. Request Flow

```text
Input       CIL bytecode plus runtime configuration
    ↓
Processing  JIT-compile on first call, manage memory, schedule async continuations
    ↓
Output      a fast, memory-safe, cross-platform application
```

## 3. Real-World Usage

**Stack Overflow** ran one of the internet's busiest sites on a handful of .NET servers. It remains a well-documented case that a well-engineered monolith on a fast runtime can outperform a far more complex distributed design.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Microsoft's bytecode runtime with JIT compilation and garbage collection |
| **Why does it exist?** | For portability, memory safety and multi-language support on one runtime |
| **Where does it belong?** | Between CIL bytecode and the operating system |
| **When should I use it?** | Long-running concurrent services, especially with C# expertise on the team |

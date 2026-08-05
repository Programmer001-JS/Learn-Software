# JVM

> **In one line —** the Java Virtual Machine: a bytecode runtime with a world-class JIT compiler and garbage collector, which is why a "slow interpreted language" ended up powering banks and Netflix.

| | |
|---|---|
| **Full name** | Java Virtual Machine |
| **Category** | Language Runtime |
| **Architectural Layer** | Runtime |
| **Languages** | Java, Kotlin, Scala, Clojure, Groovy |
| **Related notes** | [Bytecode](Bytecode.md) · [Virtual Machines](Virtual%20Machines.md) · [Runtime Explained](Runtime%20Explained.md) · [Spring Boot](../06%20-%20Backend%20Architecture/Spring%20Boot.md) · [Thread](../02%20-%20Computer%20Science%20Fundamentals/Thread.md) |

---

## 1. Short Definition

*What is it?*

The JVM is a [process virtual machine](Virtual%20Machines.md) that executes Java [bytecode](Bytecode.md). It interprets at first, then JIT-compiles frequently executed methods to machine code, and manages memory with a highly tuned garbage collector.

---

## 2. Purpose

*What is its main purpose?*

To run the same compiled artifact on any platform, at close to native speed, with automatic memory management and real multithreading.

---

## 3. Problem

*What engineering problem does it solve?*

Two problems at once: C and C++ programs had to be compiled per platform and were riddled with memory-safety bugs. The JVM removed both — one `.jar` everywhere, and no manual memory management.

---

## 4. Architecture Position

```text
Java / Kotlin / Scala source
    ↓  javac / kotlinc
Bytecode (.class, .jar)
    ↓
┌─────────────── JVM ───────────────┐
│  Class loader                     │
│  Bytecode verifier                │
│  Interpreter → JIT (C1, C2)       │
│  Garbage collector (G1, ZGC)      │
│  Thread scheduler                 │
└────────────────┬──────────────────┘
                 ↓
       Operating system → hardware
```

---

## 5. Real World Example

- **Netflix** — the majority of its backend microservices run on the JVM.
- **Banks and payment systems** — long-lived JVM services where throughput and stability outweigh startup time.
- **Kafka, Elasticsearch, Cassandra, Spark, Hadoop** — the modern data infrastructure stack is largely JVM software.
- **Android** — runs a JVM-family runtime (ART) with ahead-of-time compilation.

---

## 6. Input

*What does it receive as input?*

Bytecode in `.class` and `.jar` files, plus configuration flags — heap size (`-Xmx`), garbage collector choice, JIT tuning.

---

## 7. Processing

*What happens inside it?*

```text
Class loaded
    ↓
Bytecode VERIFIED  (rejects malformed or unsafe bytecode)
    ↓
Interpreted at first
    ↓
Method called thousands of times → "hot"
    ↓
C1 compiler: quick compilation
    ↓
Still hot → C2 compiler: aggressive optimisation
    ↓
Assumptions break → deoptimise, back to interpreting
```

> [!IMPORTANT]
> This tiered approach is why the JVM starts slowly and then becomes fast. Benchmarking a JVM service without warming it up first produces meaningless numbers.

---

## 8. Output

*What does it return?*

A running application with heavily optimised machine code for its hot paths, and automatically managed memory.

---

## 9. Garbage collection

The JVM's collectors are among the most sophisticated in existence, and choosing between them is a genuine engineering decision:

| Collector | Optimises for |
|---|---|
| **G1** | Balanced throughput and pause time — the default |
| **ZGC** | Very low pause times, even on huge heaps |
| **Shenandoah** | Similar goals to ZGC |
| **Parallel** | Maximum throughput, longer pauses |

> [!TIP]
> If your average latency is good but p99 is bad, look at GC pauses before looking at your code.

---

## 10. Communication

- **Operating system** — through system calls
- **Native code** — via JNI or the modern Foreign Function & Memory API
- **Other JVMs** — over the network like any service
- **Monitoring tools** — through JMX, which is one of the best runtime observability stories anywhere

---

## 11. Dependencies

- A JDK or JRE installed, or bundled into the container image
- Sufficient memory — the JVM reserves a heap up front
- A matching bytecode version; newer bytecode will not load on an older JVM

---

## 12. Alternatives

```text
JVM              portable, fast after warm-up, heavy startup
    ↓
GraalVM Native   compile ahead of time to a native binary: millisecond startup, less peak throughput
    ↓
.NET             very similar architecture and trade-offs
    ↓
Go               compiled, tiny binary, simpler runtime, weaker JIT-level optimisation
```

---

## 13. When To Use

> [!TIP]
> Use the JVM for **long-running, high-throughput services** where startup time is amortised over days: APIs, data processing, streaming systems, anything where the ecosystem (Spring, Kafka, Spark) is a major advantage.

---

## 14. When NOT To Use

> [!CAUTION]
> - **Serverless and CLI tools** — a 1–3 second startup is fatal for a function billed by the millisecond. Use GraalVM native images or a different language.
> - **Memory-constrained environments** — the JVM baseline is hundreds of megabytes.
> - **Small scripts** — the ceremony far outweighs the task.

---

## 15. Advantages

- Near-native speed after JIT warm-up
- Excellent garbage collectors with tunable pause behaviour
- Genuine multithreading with no GIL
- Outstanding observability and profiling tooling
- Enormous, mature ecosystem and very strong backwards compatibility

---

## 16. Disadvantages

- Slow startup — seconds, not milliseconds
- Large memory footprint
- GC pauses affect tail latency
- Large container images
- Substantial tuning surface; the defaults are not always right

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Near-native after warm-up; sometimes beats C on branch-heavy code |
| **Memory** | 100 MB+ baseline before your application allocates anything |
| **CPU** | JIT compilation and GC run on background threads |
| **Startup** | 1–3 seconds typical; ~50 ms with a GraalVM native image |
| **Container** | 200–400 MB images unless carefully trimmed |

---

## 18. Security Considerations

> [!CAUTION]
> **Log4Shell (CVE-2021-44228)** was a JVM-ecosystem vulnerability that allowed remote code execution through a logged string. It is the clearest possible demonstration that your dependencies are your attack surface.

- **Deserialisation of untrusted data** is a long-standing and severe Java vulnerability class
- **Bytecode verification** is a real defence — the JVM rejects malformed bytecode before running it
- Bytecode decompiles very cleanly; obfuscation raises effort but does not hide logic
- Keep the JDK patched; it is a large native codebase with regular security releases

---

## 19. Mental Model

> [!NOTE]
> **The JVM is a professional kitchen that takes twenty minutes to heat up.**
>
> Useless if you want one sandwich. Once the ovens are hot, it serves five hundred covers a night at a consistency a home kitchen cannot match — and it cleans up after itself, occasionally pausing service to do so.

---

## 20. Mini Architecture Diagram

```text
.java / .kt
    ↓ compile
.class / .jar  (bytecode)
    ↓
Class loader → Verifier
    ↓
Interpreter → C1 JIT → C2 JIT
    ↓
Machine code
    ↓
Heap  ←── Garbage collector (G1 / ZGC)
    ↓
Operating system → hardware
```

---

## 21. Complete Request Flow

A Spring Boot service handling a request:

```text
JVM starts, loads classes, initialises the heap   (~2 s, once)
    ↓
First requests: interpreted, relatively slow
    ↓
After a few thousand requests, hot methods are JIT-compiled
    ↓
Request arrives → thread taken from the pool
    ↓
Handler runs as optimised machine code
    ↓
Objects allocated on the heap
    ↓
Response returned; thread goes back to the pool
    ↓
Periodically: GC reclaims garbage, occasionally pausing briefly
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> The JVM trades startup time and memory for near-native throughput, real threading and industrial-grade tooling — an excellent bargain for long-running services and a poor one for anything short-lived.

---

## 23. Common Mistakes

- **Benchmarking without warm-up** — the first thousand iterations are not representative
- **Not setting `-Xmx`** in containers, so the JVM misreads available memory and gets OOM-killed
- **Ignoring GC logs** when investigating tail latency
- **Using the JVM for serverless** without a native image
- **Deserialising untrusted data**
- **Assuming more heap is always better** — larger heaps can mean longer pauses

---

## 24. Open Source Technologies

- **OpenJDK** — the reference implementation
- **GraalVM** — ahead-of-time native compilation for JVM languages
- **Spring Boot**, **Quarkus**, **Micronaut** — application frameworks
- **Kafka**, **Elasticsearch**, **Cassandra**, **Spark** — major JVM infrastructure
- **async-profiler**, **JFR**, **VisualVM** — profiling and diagnostics

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Start a JVM application and measure request latency for the first 100 requests versus requests 10,000–10,100.
- [ ] Enable GC logging on a service and find the longest pause.
- [ ] Explain in two sentences why the JVM is a poor fit for AWS Lambda without a native image.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Java / Kotlin source
    ↓
Bytecode
    ↓
JVM (verifier → interpreter → JIT → GC)
    ↓
Machine code
    ↓
Operating system → hardware
```

## 2. Request Flow

```text
Input       bytecode plus runtime configuration
    ↓
Processing  verify → interpret → JIT-compile hot methods → collect garbage
    ↓
Output      a fast, memory-managed, portable running application
```

## 3. Real-World Usage

**Netflix** runs the bulk of its backend on the JVM. For services that stay up for weeks and handle enormous request volumes, slow startup costs nothing and the JIT plus mature tooling pay for themselves continuously.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A bytecode runtime with a tiered JIT and advanced garbage collectors |
| **Why does it exist?** | For portability and memory safety without giving up speed |
| **Where does it belong?** | Between JVM-language bytecode and the operating system |
| **When should I use it?** | Long-running, high-throughput services — not short-lived processes |

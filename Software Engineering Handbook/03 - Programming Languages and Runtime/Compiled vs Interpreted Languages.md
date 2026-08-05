# Compiled vs Interpreted Languages

> **In one line —** compiled languages translate your whole program to machine code before running it; interpreted ones translate it while running — and almost every modern language is somewhere in between.

| | |
|---|---|
| **Category** | Foundation Concept |
| **Architectural Layer** | Language / Runtime |
| **Related notes** | [Bytecode](Bytecode.md) · [Virtual Machines](Virtual%20Machines.md) · [Runtime Explained](Runtime%20Explained.md) · [Binary and Machine Code](../02%20-%20Computer%20Science%20Fundamentals/Binary%20and%20Machine%20Code.md) |

---

## 1. Short Definition

*What is it?*

A **compiled** language is translated to [machine code](../02%20-%20Computer%20Science%20Fundamentals/Binary%20and%20Machine%20Code.md) ahead of time, producing a binary the CPU runs directly. An **interpreted** language is read and executed line by line at runtime by another program — the interpreter.

---

## 2. Purpose

*What is its main purpose?*

To bridge the gap between how humans want to write code and the only thing a [CPU](../02%20-%20Computer%20Science%20Fundamentals/CPU.md) understands. The two approaches trade **speed** against **flexibility and development velocity**.

---

## 3. Problem

*What engineering problem does it solve?*

A CPU executes only machine code, which is unwritable by humans at any scale. Every language needs a translation strategy, and there are only two basic options: translate everything first, or translate as you go.

---

## 4. Architecture Position

```text
COMPILED                          INTERPRETED

Source code                       Source code
    ↓  compiler (once)                ↓
Machine code (binary)             Interpreter reads it line by line
    ↓                                 ↓
CPU runs it directly              Interpreter executes each statement
                                      ↓
                                  CPU runs the INTERPRETER
```

> [!IMPORTANT]
> In the interpreted case, the CPU is not running your program — it is running the interpreter, which is *simulating* your program. That indirection is the whole source of the performance difference.

---

## 5. The spectrum in practice

Almost nothing is purely one or the other any more.

| Language | Model |
|---|---|
| **C, C++, Rust, Go** | Fully compiled to machine code ahead of time |
| **Java, C#** | Compiled to [bytecode](Bytecode.md), then JIT-compiled to machine code at runtime |
| **Python** | Compiled to bytecode, then interpreted by [CPython](CPython.md) |
| **JavaScript** | Interpreted, then aggressively JIT-compiled by V8 |
| **TypeScript** | Transpiled to JavaScript first — a source-to-source compiler |

---

## 6. Just-In-Time compilation

JIT is the modern compromise, and it is why "interpreted languages are slow" is now a much weaker claim than it used to be.

```text
Program starts → interpret (fast to start)
    ↓
Runtime notices a function is called thousands of times  ("hot")
    ↓
JIT compiles that function to machine code
    ↓
Subsequent calls run at near-compiled speed
```

> [!TIP]
> A JIT can sometimes beat an ahead-of-time compiler, because it sees the *actual* data and branch behaviour at runtime, not just what the source suggests.

---

## 7. Real World Example

- **Go** compiles to a single static binary — one of the main reasons it became popular for containers and CLI tools.
- **Python** dominates data science and scripting because the write-run-fix loop has no compile step, and the slow parts (NumPy, PyTorch) are C underneath.
- **JavaScript** is the same language in a browser and on a server precisely because it does not need to be compiled for a target machine.
- **Java** runs on any platform with a JVM — "write once, run anywhere" is the bytecode model as a product promise.

---

## 8. Input and Output

| | Compiled | Interpreted |
|---|---|---|
| **Input** | Source code, once | Source code, every run |
| **Output** | A binary for one OS + CPU | Nothing persistent — execution happens directly |
| **Errors found** | Many at compile time | Mostly at runtime, on the line that executes |

---

## 9. Internal Idea

*How does it work internally?*

Compilation is **translation**; interpretation is **simulation**.

A compiler can afford to spend seconds analysing your entire program: inlining functions, removing dead code, allocating [registers](../02%20-%20Computer%20Science%20Fundamentals/Registers.md) optimally. An interpreter must decide what to do next, right now, thousands of times per second — leaving no time for that kind of analysis.

---

## 10. Comparison

| | Compiled | Interpreted |
|---|---|---|
| **Speed** | Fast | 10–100× slower without a JIT |
| **Startup** | Instant | Interpreter must start first |
| **Portability** | One binary per platform | Runs anywhere the interpreter exists |
| **Error detection** | Compile time | Runtime |
| **Development loop** | Compile, then run | Run immediately |
| **Deployment** | Ship one binary | Ship source plus a runtime |
| **Memory** | Low overhead | Interpreter overhead per process |

---

## 11. When To Use

> [!TIP]
> **Compiled** — when speed, low memory, or a single-file deployment matters: system tools, high-throughput services, CLI utilities, anything CPU-bound.
>
> **Interpreted** — when development speed and flexibility matter more: web backends, scripts, data analysis, glue code. Most web applications are I/O-bound, so the language's raw speed is rarely the bottleneck.

---

## 12. When NOT To Use

> [!CAUTION]
> - Do not choose a compiled language "for performance" when your bottleneck is a missing database index. Measure first.
> - Do not choose an interpreted language for tight numeric loops without a native library underneath.
> - Do not underestimate deployment: shipping a Python service means shipping a specific Python version, its dependencies and a container to hold them.

---

## 13. Advantages and Disadvantages

**Compiled**
- Fastest execution, lowest memory overhead
- Errors caught before shipping
- Single self-contained binary
- ✗ Slower development loop, one build per platform

**Interpreted**
- Immediate feedback, no build step
- Runs anywhere the interpreter runs
- Dynamic behaviour, introspection, hot reloading
- ✗ Slower, higher memory, errors surface in production

---

## 14. Performance Impact

| Resource | Compiled | Interpreted |
|---|---|---|
| **Speed** | Baseline | 10–100× slower, much less with a JIT |
| **Memory** | Low | Interpreter and objects add substantial overhead |
| **CPU** | Efficient | Interpretation overhead on every operation |
| **Startup** | ~1 ms | ~50–500 ms |
| **Container size** | ~10 MB (Go) | ~100–1000 MB (Python with dependencies) |

---

## 15. Security Considerations

> [!CAUTION]
> Interpreted languages can execute code constructed at runtime — `eval()`, `exec()`, `pickle`, dynamic imports. Passing user input anywhere near those is remote code execution, and it is one of the most severe vulnerability classes there is.

- Compiled binaries can still be reverse-engineered; compilation is not obfuscation
- Interpreted deployments ship source code, so secrets in source are shipped too
- Both are equally exposed to supply-chain attacks through dependencies

---

## 16. Mental Model

> [!NOTE]
> **Compiling = translating a whole book before anyone reads it.** Slow to prepare, fast to read, and fixed once printed.
>
> **Interpreting = a live translator standing beside you.** You can start immediately and change direction at any time, but every sentence goes through them.

---

## 17. Mini Architecture Diagram

```text
Source code
    ↓
┌───────────────┬────────────────────┬──────────────────┐
│  COMPILED     │  BYTECODE + VM     │  INTERPRETED     │
│  C, Go, Rust  │  Java, C#, Python  │  Bash, classic JS│
│      ↓        │        ↓           │        ↓         │
│ machine code  │  bytecode → JIT    │  read line by line│
│      ↓        │        ↓           │        ↓         │
│     CPU       │       CPU          │  interpreter→CPU │
└───────────────┴────────────────────┴──────────────────┘
```

---

## 18. Complete Request Flow

The same line of code, three ways:

```text
COMPILED (Go)
  x := a + b  →  compiled once to ADD instructions  →  CPU executes

BYTECODE + JIT (Java)
  x = a + b   →  bytecode IADD  →  JVM interprets, then JIT-compiles the hot method  →  CPU

INTERPRETED (Python)
  x = a + b   →  bytecode BINARY_ADD  →  CPython loop: fetch opcode,
                 check both types, call the right add function, allocate a result object
```

> [!IMPORTANT]
> The Python version does dozens of machine instructions where Go does one. That is the entire difference, and it is also why NumPy exists — it moves the loop into C.

---

## 19. Key Takeaway

> [!IMPORTANT]
> Compiled trades development speed for execution speed, interpreted trades the reverse — and modern JIT runtimes have blurred the line enough that the choice is usually about ecosystem and deployment, not raw performance.

---

## 20. Common Mistakes

- **Choosing a language for speed** without measuring where the time actually goes
- **Believing interpreted always means slow** — V8 and the JVM are extraordinary pieces of engineering
- **Ignoring startup time** in serverless, where cold starts are billed and user-visible
- **Forgetting the runtime is part of the deployment** for interpreted languages
- **Assuming a compiled binary hides your logic**

---

## 21. Open Source Technologies

- **GCC**, **LLVM/Clang** — the major compiler toolchains
- **CPython**, **PyPy**, **V8**, **HotSpot JVM** — runtimes worth understanding
- **GraalVM** — compiles JVM languages ahead of time to native binaries
- **WebAssembly** — a compiled target that runs in the browser sandbox

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Write the same loop in Python and in Go, time both, and explain the ratio you observe.
- [ ] Measure your application's startup time and decide whether it matters for how you deploy.
- [ ] Explain in two sentences what a JIT does and why it can beat an ahead-of-time compiler.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Source code
    ↓
Compiler  /  Interpreter  /  Bytecode + VM
    ↓
Machine code
    ↓
CPU
```

## 2. Request Flow

```text
Input       source code
    ↓
Processing  translated ahead of time, or interpreted (and possibly JIT-compiled) at runtime
    ↓
Output      machine instructions the CPU executes
```

## 3. Real-World Usage

**Docker's ecosystem** favoured Go heavily because a compiled static binary needs no runtime in the image. The same property is why a Go container is often ~10 MB while an equivalent Python one is several hundred.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Two strategies for turning source code into executable instructions |
| **Why does it exist?** | Because CPUs run only machine code, and humans cannot write it |
| **Where does it belong?** | Between your source and the CPU |
| **When should I use it?** | Compiled for speed and deployment simplicity; interpreted for development velocity |

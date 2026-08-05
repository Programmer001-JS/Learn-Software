# Bytecode

> **In one line —** an intermediate instruction set that is not tied to any CPU, so one compiled artifact can run anywhere a virtual machine exists.

| | |
|---|---|
| **Category** | Language Runtime Concept |
| **Architectural Layer** | Between source code and machine code |
| **Related notes** | [Virtual Machines](Virtual%20Machines.md) · [Compiled vs Interpreted Languages](Compiled%20vs%20Interpreted%20Languages.md) · [JVM](JVM.md) · [CPython](CPython.md) · [Binary and Machine Code](../02%20-%20Computer%20Science%20Fundamentals/Binary%20and%20Machine%20Code.md) |

---

## 1. Short Definition

*What is it?*

Bytecode is a compact set of instructions for an **imaginary computer** — a [virtual machine](Virtual%20Machines.md) — rather than for a real CPU. It sits between human-readable source code and CPU-specific [machine code](../02%20-%20Computer%20Science%20Fundamentals/Binary%20and%20Machine%20Code.md).

---

## 2. Purpose

*What is its main purpose?*

To decouple compilation from the target hardware. Compile once to bytecode, then run it on any machine that has a virtual machine for it — x86, ARM, Windows, Linux, whatever.

---

## 3. Problem

*What engineering problem does it solve?*

Machine code is not portable: a binary compiled for x86-64 Linux will not run on ARM macOS.

```text
WITHOUT bytecode          WITH bytecode

source                    source
  ↓ compile per target        ↓ compile ONCE
x86 binary                bytecode
ARM binary                    ↓
Windows binary            JVM / CLR / CPython on any platform
macOS binary
```

It also gives the runtime a stable, analysable format it can optimise at runtime — which is what makes JIT compilation practical.

---

## 4. Architecture Position

```text
Source code   (Java, C#, Python)
    ↓  compiler — happens once
BYTECODE      (.class, .dll, .pyc)      ← you are here
    ↓
Virtual machine  (JVM, CLR, CPython)
    ↓  interpret and/or JIT-compile
Machine code
    ↓
CPU
```

---

## 5. Real World Example

- **Java `.class` files** — the entire "write once, run anywhere" promise is bytecode plus a JVM on every platform.
- **Python `.pyc` files** in `__pycache__` — Python compiles your source to bytecode on first import and caches it, which is why the second start of a large application is faster.
- **.NET assemblies** — `.dll` files contain CIL bytecode, not machine code, which is why the same assembly runs on Windows, Linux and macOS.
- **WebAssembly** — a portable bytecode designed for the browser sandbox, now used well beyond it.

---

## 6. Input

*What does it receive as input?*

Source code processed by a compiler — `javac`, `csc`, or CPython's internal compiler.

---

## 7. Processing

*What happens inside it?*

Bytecode is usually **stack-based**: instructions push and pop values on a virtual stack rather than naming [registers](../02%20-%20Computer%20Science%20Fundamentals/Registers.md). This makes it simple and hardware-independent.

```text
Source:    x = a + b

Python bytecode:
    LOAD_NAME    a          push a onto the stack
    LOAD_NAME    b          push b
    BINARY_ADD              pop two, add, push the result
    STORE_NAME   x          pop into x
```

You can see this yourself: `python -m dis your_file.py`.

---

## 8. Output

*What does it return?*

A portable artifact — `.class`, `.pyc`, `.dll`, `.wasm` — that a virtual machine can load and execute.

---

## 9. Internal Idea

*How does it work internally?*

Bytecode is designed for a machine that does not exist, chosen precisely so it is easy to interpret, easy to verify, and easy to translate into real machine code later.

> [!IMPORTANT]
> Because it is analysable, the runtime can watch which parts run most often and **JIT-compile only those**. That is why the JVM and V8 reach near-native speed while keeping portability.

---

## 10. Communication

- **Compiler** above it — produces it
- **[Virtual machine](Virtual%20Machines.md)** below it — executes or compiles it
- **JIT compiler** — turns hot bytecode into machine code at runtime
- **Verifiers** — check bytecode is well-formed before running it, a real security feature

---

## 11. Dependencies

- A **virtual machine** for the target platform; bytecode alone runs nowhere
- A matching version — bytecode compiled by a newer compiler often will not load in an older VM

---

## 12. Alternatives

```text
Machine code    fastest, not portable
    ↓
Bytecode + VM   portable, JIT-optimisable, needs a runtime installed
    ↓
Source shipped  maximally flexible, slowest start, source is visible
    ↓
WebAssembly     portable bytecode with a strong sandbox
```

---

## 13. When To Use

> [!TIP]
> You use bytecode implicitly by choosing Java, C#, Python or Kotlin. Choose those when platform portability and a large ecosystem matter more than startup time and container size.

---

## 14. When NOT To Use

> [!CAUTION]
> Avoid bytecode runtimes when startup latency dominates — serverless cold starts are the classic case — or when you need a small, self-contained deployment artifact. A Go binary in a 10 MB container starts in milliseconds; a JVM service in a 400 MB image takes seconds.

---

## 15. Advantages

- One build artifact for every platform
- Enables JIT optimisation using real runtime behaviour
- Smaller and faster to load than parsing source every time
- Can be verified before execution, which blocks whole classes of malformed input

---

## 16. Disadvantages

- Requires a runtime installed or bundled
- Slower startup than a native binary
- Easily decompiled — bytecode retains far more structure than machine code
- Version coupling between compiler and VM

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Slower than native when interpreted; near-native once JIT-compiled |
| **Memory** | The VM adds a baseline footprint (tens to hundreds of MB for a JVM) |
| **CPU** | JIT compilation itself consumes CPU during warm-up |
| **Startup** | Noticeably slower — the VM must start and warm up |
| **Container size** | The runtime must ship with the application |

---

## 18. Security Considerations

> [!CAUTION]
> Bytecode **decompiles remarkably well**. Java and .NET bytecode can be turned back into near-original source with free tools. Never assume shipping compiled bytecode hides secrets, keys or business logic.

- **Bytecode verification** in the JVM and CLR is a genuine security feature — it rejects malformed code before it runs
- **Obfuscators** (ProGuard, R8) raise the effort required but do not make decompilation impossible
- Loading bytecode from an untrusted source is equivalent to running untrusted code

---

## 19. Mental Model

> [!NOTE]
> **Bytecode is a recipe written in a universal notation.**
>
> It is not written for any specific kitchen. Any cook who understands the notation can follow it, adapting it to their own stove — which is exactly what a virtual machine does.

---

## 20. Mini Architecture Diagram

```text
Java source     Python source     C# source
    ↓ javac         ↓ compile         ↓ csc
  .class            .pyc              .dll
    ↓                ↓                 ↓
   JVM            CPython             CLR
    ↓                ↓                 ↓
        machine code → CPU
```

---

## 21. Complete Request Flow

Running a Python file:

```text
python app.py
    ↓
Is there a cached .pyc newer than the source?
    ↓ NO → compile source to bytecode, cache it in __pycache__
    ↓ YES → load the cached bytecode
    ↓
CPython's evaluation loop:
    fetch opcode → decode → execute → next
    ↓
Each opcode runs C code inside the interpreter
    ↓
CPU executes the interpreter, which simulates your program
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> Bytecode is portable instructions for an imaginary machine — it buys you "compile once, run anywhere" and pays for it with a runtime you must ship and a slower start.

---

## 23. Common Mistakes

- **Confusing bytecode with machine code** — bytecode cannot run without a VM
- **Assuming it hides your source** — decompilers are excellent
- **Committing `__pycache__` or `bin/` to git** — build output does not belong in version control
- **Version mismatches** between compiler and runtime, producing confusing load errors
- **Ignoring JIT warm-up** when benchmarking — the first thousand iterations are not representative

---

## 24. Open Source Technologies

- **OpenJDK** — the reference JVM
- **CPython**, **PyPy** — Python bytecode runtimes
- **.NET runtime** — open source CLR
- **WebAssembly** with **Wasmtime**, **Wasmer** — portable bytecode outside the browser
- **javap**, **dis**, **ILSpy** — inspect bytecode directly

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Run `python -m dis` on a five-line function and read the bytecode it produces.
- [ ] Delete `__pycache__` and compare first and second startup times.
- [ ] Explain in two sentences why Java can run on both a phone and a server without recompiling.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Source code
    ↓
Bytecode
    ↓
Virtual machine (interpret + JIT)
    ↓
Machine code
    ↓
CPU
```

## 2. Request Flow

```text
Input       source code
    ↓
Processing  compiled once to portable bytecode; VM interprets and JIT-compiles hot paths
    ↓
Output      machine instructions executed by the CPU
```

## 3. Real-World Usage

**Android** ships application code as bytecode (DEX), which the device compiles to native code at install time. One uploaded artifact runs across thousands of different phone processors — bytecode is what makes that possible.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Portable instructions for a virtual machine rather than a real CPU |
| **Why does it exist?** | Because machine code is tied to one processor architecture |
| **Where does it belong?** | Between the compiler and the runtime |
| **When should I use it?** | When portability and ecosystem outweigh startup time and artifact size |

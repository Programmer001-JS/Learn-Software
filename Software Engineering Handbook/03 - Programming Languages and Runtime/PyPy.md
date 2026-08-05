# PyPy

> **In one line —** an alternative Python implementation with a JIT compiler that makes long-running pure-Python code several times faster — at the cost of C-extension compatibility.

| | |
|---|---|
| **Category** | Language Runtime |
| **Architectural Layer** | Runtime |
| **Written in** | RPython (a restricted subset of Python) |
| **Related notes** | [CPython](CPython.md) · [Runtime Explained](Runtime%20Explained.md) · [Bytecode](Bytecode.md) · [Compiled vs Interpreted Languages](Compiled%20vs%20Interpreted%20Languages.md) |

---

## 1. Short Definition

*What is it?*

PyPy is a Python interpreter that includes a **just-in-time compiler**. It watches which code runs often and compiles those parts to machine code, so hot loops execute at a speed [CPython](CPython.md) cannot approach.

---

## 2. Purpose

*What is its main purpose?*

To make pure-Python code fast without asking you to rewrite it in C. Same language, same source files, different runtime.

---

## 3. Problem

*What engineering problem does it solve?*

CPython interprets every operation individually, costing roughly 50–100 machine instructions per simple Python operation. For CPU-bound Python that overhead dominates everything.

```text
CPython   interpret every iteration, every time
PyPy      interpret at first, then compile the hot loop to machine code
```

---

## 4. Architecture Position

```text
Your Python code  (unchanged)
    ↓
PyPy: parse → bytecode
    ↓
Interpret first  →  detect hot loops  →  JIT-compile them to machine code
    ↓
Machine code executes directly
    ↓
Operating system → hardware
```

---

## 5. How the JIT works

```text
Loop runs 1st–999th time    → interpreted, and traced
    ↓
Runtime observes: "these are always integers, this branch is never taken"
    ↓
Compile a specialised machine-code version with those assumptions
    ↓
Loop runs 1000th+ time      → native speed
    ↓
An assumption breaks?  → discard and fall back to interpreting
```

> [!IMPORTANT]
> This is called **tracing JIT**. It can produce code faster than a static compiler would, because it optimises for what actually happens rather than for every case that might.

---

## 6. Real World Example

- **Long-running simulations and numeric algorithms** written in pure Python — often 4–10× faster with no code changes.
- **Quantitative and financial backtesting**, where the same loop runs for hours.
- **PyPy is generally not used** for typical Django or FastAPI services, because those are I/O-bound and depend heavily on C extensions.

---

## 7. Input, Processing and Output

**Input:** ordinary `.py` files — no annotations, no changes.

**Processing:** interpret, trace, compile hot paths, deoptimise when assumptions fail.

**Output:** identical program behaviour, substantially faster for CPU-bound pure Python.

---

## 8. Internal Idea

*How does it work internally?*

PyPy is written in **RPython**, a restricted Python subset from which a JIT-equipped interpreter can be generated automatically. The insight is that the JIT was not hand-written for Python — it was *generated* from a description of the interpreter, which is why the same toolchain has produced JITs for other languages.

---

## 9. Communication

- **Your Python code** — unchanged
- **C extensions** — through `cpyext`, a compatibility layer that works but is often **slower** than on CPython
- **The operating system** — as any runtime does

---

## 10. Dependencies

- A PyPy build matching your Python version (PyPy tracks specific CPython versions)
- Dependencies that either are pure Python or have working C-extension support

---

## 11. Alternatives

```text
CPython      maximum compatibility, moderate speed        ← the default
    ↓
PyPy         4–10× on pure Python, weaker C-extension story
    ↓
Cython       compile annotated Python to C — targeted, keeps CPython
    ↓
Numba        JIT numeric functions with a decorator
    ↓
Rust / C extension   rewrite the hot 5%
```

---

## 12. When To Use

> [!TIP]
> Use PyPy when your bottleneck is **pure-Python CPU work in a long-running process** and your dependencies do not lean heavily on C extensions. Measure your actual workload — the published benchmarks may not resemble it.

---

## 13. When NOT To Use

> [!CAUTION]
> - **Heavy NumPy, PyTorch, pandas or psycopg use** — those are already C, and PyPy's compatibility layer can make them slower.
> - **I/O-bound web applications** — the interpreter is not the bottleneck, so you gain nothing.
> - **Short-lived scripts and serverless functions** — the JIT never warms up, and startup is slower.
> - **When you need the very latest Python version** — PyPy typically lags behind.

---

## 14. Advantages

- Often 4–10× faster on CPU-bound pure Python
- No source changes required
- A better garbage collector than reference counting for allocation-heavy workloads
- Fully open source and mature

---

## 15. Disadvantages

- C-extension compatibility is the recurring, practical problem
- Higher memory usage — the JIT stores compiled code and metadata
- Slower startup and a warm-up period before the speedup arrives
- Trails CPython in supported language versions
- Still has a GIL, so it does not solve CPU parallelism either

---

## 16. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | 4–10× on hot pure-Python loops; no gain, or a loss, elsewhere |
| **Memory** | Higher than CPython — compiled code plus JIT metadata |
| **CPU** | JIT compilation consumes CPU during warm-up |
| **Startup** | Slower, and the benefit only appears after warm-up |
| **Concurrency** | Unchanged — the GIL is still there |

---

## 17. Security Considerations

> [!CAUTION]
> PyPy is a smaller project with fewer eyes on it than CPython, and security patches can arrive later. For a service exposed to the internet, that difference in patch cadence is a real consideration.

The language-level risks are identical: `pickle`, `eval` and unsafe deserialisation are exactly as dangerous here.

---

## 18. Mental Model

> [!NOTE]
> **CPython reads the recipe aloud every single time it cooks the dish.**
>
> **PyPy reads it aloud a thousand times, then memorises it** — after which it cooks at full speed. Excellent if you are cooking all day; pointless if you cook once and leave.

---

## 19. Mini Architecture Diagram

```text
Python source
    ↓
PyPy bytecode
    ↓
Interpreter  ──(hot loop detected)──►  Tracing JIT
    ↓                                       ↓
slow path                            machine code (fast path)
    ↓                                       ↓
             Operating system → CPU
```

---

## 20. Complete Request Flow

A long-running computation:

```text
Process starts (slower than CPython)
    ↓
Loop iterations 1–1000: interpreted and traced
    ↓
JIT compiles the loop with observed type assumptions
    ↓
Iterations 1000+: native machine code, several times faster
    ↓
An input of an unexpected type appears
    ↓
Guard fails → deoptimise → back to interpreting → retrace
```

---

## 21. Key Takeaway

> [!IMPORTANT]
> PyPy makes long-running pure-Python code several times faster with no code changes — but only if C extensions are not your bottleneck, and only after the JIT has warmed up.

---

## 22. Common Mistakes

- **Expecting a speedup on I/O-bound web applications**
- **Using it with heavy NumPy or pandas workloads**, where it can be slower
- **Benchmarking short scripts**, where the JIT never warms up
- **Assuming it removes the GIL** — it does not
- **Switching for performance without measuring** where the time actually goes

---

## 23. Open Source Technologies

- **PyPy** — the runtime itself
- **Cython**, **Numba**, **mypyc** — targeted alternatives that keep CPython
- **CPython 3.13+ free-threaded builds** — ongoing work to remove the GIL in CPython itself

---

## 24. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 25. Workbook Exercise

- [ ] Write a CPU-heavy pure-Python loop and time it on CPython and PyPy.
- [ ] Add a NumPy version of the same computation and compare all three.
- [ ] Decide, in two sentences, whether PyPy would help your current project — and say why.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Python code
    ↓
PyPy interpreter + tracing JIT
    ↓
Machine code
    ↓
Operating system → CPU
```

## 2. Request Flow

```text
Input       ordinary Python source
    ↓
Processing  interpret → trace hot loops → JIT-compile → deoptimise if assumptions break
    ↓
Output      identical behaviour, much faster for CPU-bound pure Python
```

## 3. Real-World Usage

PyPy is used mainly in **scientific and quantitative computing**, where the same pure-Python loop runs for hours and the warm-up cost is irrelevant compared with the total runtime.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A Python implementation with a tracing JIT compiler |
| **Why does it exist?** | Because CPython's per-operation overhead dominates CPU-bound Python |
| **Where does it belong?** | As a drop-in replacement for the CPython runtime |
| **When should I use it?** | Long-running, CPU-bound, pure-Python workloads without heavy C extensions |

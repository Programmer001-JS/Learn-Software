# CPython

> **In one line —** the reference Python implementation: written in C, easy to extend, and deliberately single-threaded for Python code because of the GIL.

| | |
|---|---|
| **Category** | Language Runtime |
| **Architectural Layer** | Runtime |
| **Written in** | C |
| **Related notes** | [Runtime Explained](Runtime%20Explained.md) · [Bytecode](Bytecode.md) · [PyPy](PyPy.md) · [Thread](../02%20-%20Computer%20Science%20Fundamentals/Thread.md) · [Gunicorn](../06%20-%20Backend%20Architecture/Gunicorn.md) · [Uvicorn](../06%20-%20Backend%20Architecture/Uvicorn.md) |

---

## 1. Short Definition

*What is it?*

CPython is the standard implementation of Python — the program you run when you type `python`. It compiles your source to [bytecode](Bytecode.md) and then executes that bytecode in an interpreter loop written in C.

---

## 2. Purpose

*What is its main purpose?*

To run Python code, and to make it trivially easy to call C libraries from Python. That second property is why Python became the language of scientific computing and AI: NumPy, PyTorch and OpenCV are C and CUDA underneath, with Python as the control layer.

---

## 3. Problem

*What engineering problem does it solve?*

Python needed an implementation that was simple, portable and — crucially — easy to extend with native code. CPython chose simplicity and C interoperability over raw speed, and that trade decided Python's entire trajectory.

---

## 4. Architecture Position

```text
Your Python code
    ↓
CPython compiler → bytecode (.pyc)
    ↓
CPython evaluation loop  (a big C switch over opcodes)
    ↓
C standard library / native extensions (NumPy, psycopg, PyTorch)
    ↓
Operating system
    ↓
Hardware
```

---

## 5. The GIL — the single most important thing to understand

> [!IMPORTANT]
> The **Global Interpreter Lock** allows only **one thread to execute Python bytecode at a time**, per process. It exists to make CPython's reference-counting memory management safe without locking every object.

```text
4-core machine, 4 Python threads doing CPU work
    ↓
GIL allows one at a time
    ↓
Result: roughly single-core speed, plus switching overhead
```

What the GIL does **not** block:

- **I/O** — the GIL is released while waiting on a database, network or disk. This is why threads still help I/O-bound Python.
- **Native extensions** — NumPy and PyTorch release the GIL during heavy C computation, so they genuinely use multiple cores.

> [!TIP]
> The practical rule: **CPU-bound Python needs processes, not threads.** This is precisely why Gunicorn runs multiple workers and why Celery uses separate processes.

---

## 6. Input and Processing

**Input:** `.py` source files, plus configuration through environment variables and command-line flags.

**Processing:**

```text
Source  →  parsed to an AST  →  compiled to bytecode  →  cached in __pycache__
    ↓
Evaluation loop:
    fetch opcode → decode → execute → repeat
    ↓
Every value is a heap-allocated PyObject with a type and a reference count
```

Memory is managed by **reference counting** — an object is freed the instant its last reference disappears — with a cycle collector for objects that reference each other.

---

## 7. Output

*What does it return?*

A running Python process, and cached `.pyc` bytecode files for faster subsequent imports.

---

## 8. Internal Idea

*How does it work internally?*

Everything is a `PyObject`. `x = 1 + 2` does not add two machine integers — it looks up both objects' types, finds their addition method, calls it, allocates a new integer object and updates reference counts.

> [!IMPORTANT]
> That is roughly 50–100 machine instructions where C would use one. It is the entire explanation for Python's speed, and the entire reason NumPy exists — NumPy moves the loop itself into C so the per-element overhead disappears.

---

## 9. Communication

- **Native extensions** through the C API — NumPy, psycopg, Pillow, PyTorch
- **The [operating system](../02%20-%20Computer%20Science%20Fundamentals/02%20-%20Operating%20Systems.md)** via system calls
- **Other processes** through `multiprocessing`, queues, sockets
- **Servers** — [Gunicorn](../06%20-%20Backend%20Architecture/Gunicorn.md) and [Uvicorn](../06%20-%20Backend%20Architecture/Uvicorn.md) run multiple CPython processes to work around the GIL

---

## 10. Dependencies

- An operating system and a C library
- A specific version — 3.11, 3.12, 3.13. **The version is part of your deployment.**
- Native extensions compiled against a matching Python version and ABI

---

## 11. Alternatives

```text
CPython      the reference; maximum compatibility, moderate speed
    ↓
PyPy         JIT-compiled; often 4–10× faster on pure Python, weaker C-extension support
    ↓
Cython       compile annotated Python to C for hot paths
    ↓
Numba        JIT-compile numeric functions to machine code
    ↓
Rust/C extension  rewrite the hot 5% and call it from Python
```

---

## 12. When To Use

> [!TIP]
> Use CPython by default. Its ecosystem compatibility is unmatched, and for the overwhelming majority of applications the bottleneck is the database or the network rather than the interpreter.

---

## 13. When NOT To Use

> [!CAUTION]
> - **CPU-bound tight loops in pure Python** — use NumPy, Cython, or a different language for that part.
> - **Multithreaded CPU parallelism** — the GIL prevents it; use processes.
> - **Very low latency or memory-constrained environments** — the per-object overhead is substantial.
> - **Large-scale serverless** where a ~50 ms cold start plus dependency loading matters.

---

## 14. Advantages

- The largest ecosystem in general-purpose programming
- Trivial integration with C, C++, Rust and CUDA
- Extremely readable, fast to develop in
- Reference counting frees memory deterministically and promptly

---

## 15. Disadvantages

- Slow for pure-Python computation
- The GIL blocks CPU parallelism within a process
- High memory overhead per object
- Dependency and packaging complexity, particularly with native extensions
- Multiple worker processes multiply the memory footprint

---

## 16. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | 10–100× slower than C for pure Python loops |
| **Memory** | Every value is a heap object; an `int` costs ~28 bytes |
| **CPU** | One core per process for Python code; native extensions can use more |
| **GPU** | Fully capable via PyTorch/CUDA — Python orchestrates, the GPU computes |
| **Startup** | ~50 ms bare; several seconds with heavy imports |

---

## 17. Security Considerations

> [!CAUTION]
> **`pickle` is remote code execution by design.** Deserialising untrusted pickled data lets an attacker run arbitrary code. The same applies to `eval`, `exec` and `yaml.load` without `SafeLoader`.

- Keep the interpreter patched; CPython has published CVEs like any large C codebase
- Dependencies from PyPI run with your process's full privileges — pin versions and audit
- Never expose stack traces to users; they reveal paths and library versions

---

## 18. Mental Model

> [!NOTE]
> **CPython is a careful, meticulous clerk who processes one form at a time.**
>
> Slower than a machine, but able to handle any form, in any format, and to phone a specialist (a C library) whenever a task needs real speed. The GIL is the rule that only one clerk may write in the ledger at a time.

---

## 19. Mini Architecture Diagram

```text
app.py
    ↓
CPython compiler → bytecode
    ↓
┌──── Evaluation loop (single-threaded, GIL-protected) ────┐
│  fetch → decode → execute → reference counting            │
└───────────────────────────┬───────────────────────────────┘
                            ↓
      C extensions (NumPy, psycopg, PyTorch) — may release the GIL
                            ↓
                  Operating system → hardware
```

---

## 20. Complete Request Flow

A FastAPI request under Gunicorn with Uvicorn workers:

```text
Request arrives
    ↓
One of N CPython worker PROCESSES accepts it   ← processes, because of the GIL
    ↓
Uvicorn's asyncio event loop schedules the coroutine
    ↓
Your handler runs Python bytecode (holding the GIL)
    ↓
Database call → GIL RELEASED → other coroutines run
    ↓
Result returns → GIL reacquired → response serialised
    ↓
Response sent; objects freed by reference counting
```

---

## 21. Key Takeaway

> [!IMPORTANT]
> CPython trades execution speed for simplicity and C interoperability — and the GIL means CPU-bound Python scales with processes, never with threads.

---

## 22. Common Mistakes

- **Using threads for CPU-bound work** and seeing no speedup
- **Writing tight numeric loops in pure Python** instead of vectorising with NumPy
- **Forgetting each worker process has its own memory** — in-process caches and globals are not shared
- **Running too many workers** and exhausting memory
- **Unpickling untrusted data**
- **Assuming `async` makes CPU work faster** — it only helps with waiting

---

## 23. Open Source Technologies

- **CPython** — the reference implementation
- **NumPy**, **PyTorch**, **Cython**, **Numba** — the escape hatches from interpreter overhead
- **Gunicorn**, **Uvicorn** — process managers that work around the GIL
- **py-spy**, **memray** — profile CPU and memory without modifying your code

---

## 24. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 25. Workbook Exercise

- [ ] Write a CPU-bound loop, run it with 4 threads and then with 4 processes. Compare the times and explain the result.
- [ ] Profile a slow function with `py-spy` and identify whether the time is in Python or in a C extension.
- [ ] Work out how many worker processes your deployment runs and how much memory that costs in total.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Python code
    ↓
Bytecode
    ↓
CPython interpreter (GIL)
    ↓
C extensions / operating system
    ↓
Hardware
```

## 2. Request Flow

```text
Input       Python source
    ↓
Processing  compiled to bytecode, executed one opcode at a time under the GIL
    ↓
Output      program behaviour, with heavy work delegated to C extensions
```

## 3. Real-World Usage

**Instagram** runs one of the largest CPython deployments in the world. They scale horizontally with many worker processes rather than threads — a direct architectural consequence of the GIL.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The reference Python implementation, written in C |
| **Why does it exist?** | To run Python simply and to integrate easily with native code |
| **Where does it belong?** | Between Python code and the operating system |
| **When should I use it?** | By default — and scale it with processes, never threads |

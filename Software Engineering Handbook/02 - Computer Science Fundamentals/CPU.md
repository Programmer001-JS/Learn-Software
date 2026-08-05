# CPU

> **In one line —** the part of the computer that actually does the work, one tiny instruction at a time, billions of times per second.

| | |
|---|---|
| **Full name** | Central Processing Unit |
| **Category** | Hardware Component |
| **Architectural Layer** | Physical / Compute |
| **Related notes** | [Registers](Registers.md) · [Cache Memory](Cache%20Memory.md) · [RAM](RAM.md) · [GPU](GPU.md) · [Thread](Thread.md) · [Context Switching](Context%20Switching.md) · [Scheduler](Scheduler.md) |

---

## 1. Short Definition

*What is it?*

The CPU is the chip that **executes instructions**. It reads a machine instruction, performs one small operation — add, compare, move, jump — and moves to the next one. Everything a computer does is made out of these operations.

---

## 2. Purpose

*What is its main purpose?*

To turn code into actual work. The CPU is the only component that *decides* anything; RAM and disks only store, networks only transport.

---

## 3. Problem

*What engineering problem does it solve?*

We need a machine that can be **reprogrammed without being rebuilt**. A calculator built in hardware does one job forever. A CPU reads its instructions from memory, which means changing the software changes the behaviour — that single idea is what makes general-purpose computing possible.

---

## 4. Architecture Position

The CPU sits at the very bottom of the software stack, under everything you write.

```text
Your code (Python, JavaScript, Java)
    ↓
Runtime / Interpreter / JIT
    ↓
Operating System  (Kernel, Scheduler)
    ↓
CPU               ← executes machine instructions
    ↓
Registers → Cache → RAM
```

---

## 5. Real World Example

*Where is it used in real-world software?*

- **Web servers** — request parsing, JSON serialisation and template rendering are all CPU work.
- **Video encoding** (YouTube, Netflix) — extremely CPU- and GPU-heavy; this is why uploads take time to process.
- **Databases** — sorting, joining and comparing rows is CPU-bound once the data is in memory.
- **Password hashing** ([Argon2](../09%20-%20Security/Argon2.md), [bcrypt](../09%20-%20Security/bcrypt.md)) — deliberately designed to burn CPU, so attackers cannot guess quickly.

---

## 6. Input

*What does it receive as input?*

- **Instructions** — machine code loaded from RAM, e.g. `ADD`, `MOV`, `CMP`, `JMP`
- **Data** — the values those instructions operate on, held in registers or memory
- **Interrupts** — signals from hardware ("a key was pressed", "the disk finished")

---

## 7. Processing

*What happens inside it?*

The CPU repeats the **instruction cycle**:

```text
FETCH       read the next instruction from memory
    ↓
DECODE      figure out what it means
    ↓
EXECUTE     perform it in the ALU (arithmetic/logic unit)
    ↓
WRITE BACK  store the result in a register or memory
```

Modern CPUs overlap these steps (**pipelining**), run several instructions at once (**superscalar**), and guess which branch will be taken (**branch prediction**) to avoid waiting.

---

## 8. Output

*What does it return?*

A result written to a [register](Registers.md), to [cache](Cache%20Memory.md), or out to [RAM](RAM.md) — plus updated CPU flags (zero, carry, overflow) that the next instruction may test.

---

## 9. Internal Idea

*How does it work internally?*

Inside are billions of transistors arranged into logic gates. Those gates implement an **ALU** (arithmetic and logic), a **control unit** (which decodes instructions and drives everything), **registers** (tiny immediate storage), and **cache** (fast local memory).

The clock ticks billions of times per second, and on each tick the machine advances one step. The CPU is not clever — it is a very simple machine repeated at an unimaginable rate.

---

## 10. Communication

*Which components does it communicate with?*

- **[RAM](RAM.md)** — over the memory bus, via the [cache](Cache%20Memory.md)
- **[Cache](Cache%20Memory.md)** and **[registers](Registers.md)** — internal, extremely fast
- **Disk, network card, GPU** — through the operating system and I/O buses
- **The [operating system](02%20-%20Operating%20Systems.md)** — which decides which [process](Process.md) gets the CPU next

---

## 11. Dependencies

- **A clock** to drive the instruction cycle
- **RAM** holding the instructions to execute
- **An operating system** to decide what runs, on multi-tasking machines
- **Power and cooling** — heat is the real limit on modern clock speeds

---

## 12. Alternatives

```text
CPU     few powerful cores, handles anything, best for sequential logic
    ↓
GPU     thousands of simple cores, only for massively parallel maths
    ↓
TPU / NPU  built specifically for neural network operations
    ↓
FPGA / ASIC  hardware configured for one task, fastest and least flexible
```

They are not really competitors — a real machine uses several, each for the work it suits.

---

## 13. When To Use

> [!TIP]
> Think about the CPU when work is **computation-bound**: parsing, sorting, compressing, encrypting, rendering. If the profiler shows CPU at 100%, more RAM will not help you.

---

## 14. When NOT To Use

> [!CAUTION]
> Most web applications are **not** CPU-bound. They spend their time waiting on the database, disk or network. Optimising CPU work in an I/O-bound system produces no measurable improvement.
>
> Massively parallel numeric work (AI training, image processing) belongs on a [GPU](GPU.md), not a CPU.

---

## 15. Advantages

- Extremely fast at sequential, branching logic
- Completely general purpose — runs any program
- Large caches make it good at irregular memory access
- Mature tooling, compilers and debuggers

---

## 16. Disadvantages

- Few cores, so limited parallelism (typically 4–64)
- Slow compared with cache when it must wait for RAM
- Clock speed has stalled — heat, not design, is the limit
- Bad at the massive parallel arithmetic that AI workloads need

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Determines raw computation throughput |
| **Memory** | Not consumed by the CPU, but cache misses stall it badly |
| **CPU** | This *is* the CPU — watch utilisation and per-core load |
| **GPU** | Offload parallel maths here instead |
| **Network** | While waiting on the network, the CPU idles — that is why async I/O exists |

> [!IMPORTANT]
> A CPU waiting for RAM is doing nothing. This is why [cache](Cache%20Memory.md)-friendly data layout can make identical code several times faster.

---

## 18. Security Considerations

> [!CAUTION]
> **Spectre** and **Meltdown** showed that CPU speculative execution could leak memory across security boundaries — hardware itself became an attack surface, and the mitigations cost real performance.

- **Timing attacks** — comparing secrets with early-exit comparison leaks information through how long it takes; use constant-time comparison for tokens and passwords
- **Shared cloud hardware** means your CPU is shared with strangers ("noisy neighbours" and side channels)

---

## 19. Mental Model

> [!NOTE]
> **CPU = a worker who follows written instructions extremely fast but can only hold a few things in their hands ([registers](Registers.md)), keeps a few more on the desk ([cache](Cache%20Memory.md)), and has to walk to the shelf ([RAM](RAM.md)) for anything else.**

---

## 20. Mini Architecture Diagram

```text
        ┌──────────── CPU ────────────┐
        │  Control Unit               │
        │      ↓                      │
        │  Registers  ←→  ALU         │
        │      ↓                      │
        │  L1 → L2 → L3 Cache         │
        └───────────┬─────────────────┘
                    ↓
                   RAM
                    ↓
                   Disk
```

---

## 21. Complete Request Flow

What happens when your code adds two numbers:

```text
Instruction fetched from RAM (or cache)
    ↓
DECODE:  "add register A to register B"
    ↓
Operands loaded into registers
    ↓
ALU performs the addition
    ↓
Result written back to a register
    ↓
Program counter moves to the next instruction
```

And what happens when the data is *not* in cache:

```text
CPU needs value X
    ↓
Check L1 → miss
    ↓
Check L2 → miss
    ↓
Check L3 → miss
    ↓
Fetch from RAM  (~100 ns — the CPU stalls for ~300 cycles)
    ↓
Value cached for next time
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> The CPU executes one simple instruction at a time, extremely fast — and it spends much of its life waiting for data to arrive from memory.

---

## 23. Common Mistakes

- **Assuming more cores means faster** — only parallelisable work benefits
- **Optimising CPU work in an I/O-bound application** — measure first
- **Ignoring cache behaviour** — the same algorithm can be several times slower with a bad memory layout
- **Believing higher GHz always wins** — architecture and cache size often matter more
- **Confusing CPU cores with threads** — see [Thread](Thread.md) and [Multithreading](Multithreading.md)

---

## 24. Open Source Technologies

- **RISC-V** — an open instruction set architecture
- **perf**, **htop**, **py-spy** — tools for measuring where CPU time actually goes
- **LLVM**, **GCC** — compilers that turn your code into CPU instructions

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Run a profiler on a slow piece of your code and find out whether it is CPU-bound or waiting on I/O.
- [ ] Draw the instruction cycle from memory.
- [ ] Explain in two sentences why a cache miss costs the CPU hundreds of wasted cycles.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Your code
    ↓
Runtime
    ↓
Operating System
    ↓
CPU
    ↓
Registers → Cache → RAM
```

## 2. Request Flow

```text
Input       an instruction plus its operands
    ↓
Processing  fetch → decode → execute → write back
    ↓
Output      a result in a register, and updated status flags
```

## 3. Real-World Usage

**Netflix** spends enormous CPU and GPU capacity encoding every title into dozens of quality variants. That work is pure computation, done once at upload time so playback can be cheap for millions of viewers.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The chip that executes machine instructions |
| **Why does it exist?** | To make a machine that can be reprogrammed rather than rebuilt |
| **Where does it belong?** | The bottom of the stack, under the operating system |
| **When should I use it?** | Think about it whenever work is computation-bound rather than I/O-bound |

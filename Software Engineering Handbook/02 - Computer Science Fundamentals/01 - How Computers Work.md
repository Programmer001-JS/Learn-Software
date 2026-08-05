# How Computers Work

> **In one line —** a computer is a machine that fetches a number, interprets it as an instruction, does one tiny thing, and repeats that billions of times per second.

| | |
|---|---|
| **Category** | Overview note *(hub for this section)* |
| **Architectural Layer** | Hardware |
| **Sub-topics** | [CPU](CPU.md) · [RAM](RAM.md) · [Cache Memory](Cache%20Memory.md) · [Registers](Registers.md) · [SSD and HDD](SSD%20and%20HDD.md) · [GPU](GPU.md) · [Binary and Machine Code](Binary%20and%20Machine%20Code.md) |

---

## 1. Why a software engineer should care

Every performance problem, every memory error and every mysterious slowdown is ultimately explained by what the hardware is physically doing. You do not need to design chips — you need to know **which operations are cheap and which are catastrophically expensive**, because that ratio drives almost every architectural decision.

---

## 2. The four things a computer does

```text
1. STORE       hold data and instructions          (RAM, disk, registers)
2. FETCH       bring an instruction to the CPU
3. EXECUTE     perform one tiny operation          (add, compare, move)
4. REPEAT      billions of times per second
```

That is genuinely all of it. Everything else — operating systems, browsers, neural networks — is built out of these four steps.

---

## 3. The main components

| Component | Role | Analogy |
|---|---|---|
| **[CPU](CPU.md)** | Executes instructions | The worker |
| **[Registers](Registers.md)** | A few bytes inside the CPU | The worker's hands |
| **[Cache Memory](Cache%20Memory.md)** | Small, very fast memory near the CPU | The desk |
| **[RAM](RAM.md)** | Working memory, fast, erased on power off | The nearby shelf |
| **[SSD / HDD](SSD%20and%20HDD.md)** | Permanent storage, slow, survives power off | The basement archive |
| **[GPU](GPU.md)** | Thousands of simple cores for parallel maths | A hall of a thousand assistants |
| **Motherboard / buses** | The wires connecting everything | The corridors |

---

## 4. The memory hierarchy — the single most important picture

Speed and size trade against each other at every level.

```text
              SIZE          SPEED           COST
Registers     ~1 KB         ~0.3 ns         extreme
    ↑
L1 Cache      ~64 KB        ~1 ns           very high
    ↑
L2 / L3 Cache ~1–64 MB      ~4–40 ns        high
    ↑
RAM           8–128 GB      ~100 ns         moderate
    ↑
SSD           0.5–8 TB      ~100 µs         low
    ↑
HDD           1–20 TB       ~10 ms          very low
    ↑
Network       unlimited     ~0.5–150 ms     varies
```

> [!IMPORTANT]
> Reading from RAM is about **100× slower** than the CPU cache. Reading from an SSD is about **1,000× slower** than RAM. A network call is about **1,000,000× slower** than a RAM read.
>
> Almost every serious optimisation in software is an attempt to move work **up** this list.

This is why [Redis](../08%20-%20Databases%20and%20Data/Redis.md) exists (move data from disk to RAM), why CPU caches matter, and why "just add another network call" is never free.

---

## 5. The instruction cycle

```text
    ┌────────────────────────────────────┐
    │                                    │
    ▼                                    │
 FETCH      get the next instruction     │
    ↓        from memory                 │
 DECODE     work out what it means       │
    ↓                                    │
 EXECUTE    do it (add, compare, jump)   │
    ↓                                    │
 WRITE BACK store the result             │
    │                                    │
    └────────────────────────────────────┘
         repeated ~3 billion times/second
```

---

## 6. From your code to the machine

```text
Python / JavaScript / Java source
    ↓  compiler or interpreter
Bytecode or machine code
    ↓
Binary — literally numbers        01001000 10001001
    ↓
CPU instructions
    ↓
Electrical signals through transistors
```

See [Binary and Machine Code](Binary%20and%20Machine%20Code.md) and [Compiled vs Interpreted Languages](../03%20-%20Programming%20Languages%20and%20Runtime/Compiled%20vs%20Interpreted%20Languages.md).

---

## 7. The full picture

```text
        ┌─────────────────────── CPU ───────────────────────┐
        │  Registers  →  L1 Cache  →  L2 Cache  →  L3 Cache │
        └───────────────────────┬───────────────────────────┘
                                │  memory bus
                                ▼
                              RAM
                                │
                                ▼
                        SSD / HDD  (permanent)
                                │
                                ▼
                       Network / Internet
```

The further down you go, the more space you get and the more time you pay.

---

## 8. Real World Example

- **Databases** are essentially elaborate machines for avoiding disk reads: indexes, buffer pools and query planners all exist because the disk is slow.
- **Video games** organise data so the CPU cache is used efficiently — the same computation can run several times faster purely from better memory layout.
- **AI training** happens on [GPUs](GPU.md) because training is millions of independent multiplications, which is exactly what thousands of simple cores are for.

---

## 9. Mental Model

> [!NOTE]
> **A computer is a workshop.**
>
> The CPU is the worker, registers are their hands, cache is the desk, RAM is the shelf behind them, the SSD is the basement archive, and the network is a warehouse in another city. Work is fast when everything needed is on the desk, and slow when the worker keeps walking to the basement.

---

## 10. Key Takeaway

> [!IMPORTANT]
> A computer does one simple thing extremely fast; performance is almost entirely about how far the data has to travel to reach it.

---

## 11. Common Mistakes

- Assuming all memory access costs the same
- Ignoring that a network call is millions of times slower than a memory read
- Believing "the hardware is fast enough" — it is, until the data layout is wrong
- Thinking a GPU makes everything faster; it only helps massively parallel work

---

## 12. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 13. Workbook Exercise

- [ ] Redraw the memory hierarchy from memory, including the rough timings.
- [ ] For your current project, name where the data physically lives at each step of one request.
- [ ] Explain to someone non-technical why RAM disappearing on power-off is not a bug.

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
CPU  ←→  Cache  ←→  RAM  ←→  Disk
```

## 2. Request Flow

```text
Input       an instruction plus its data
    ↓
Processing  fetch → decode → execute → write back
    ↓
Output      a result in a register, cache, RAM or on disk
```

## 3. Real-World Usage

Every system in this handbook runs on this model. **PostgreSQL** keeps a buffer pool in RAM specifically to avoid disk reads, which is the memory hierarchy turned into a product feature.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A machine that fetches, decodes and executes instructions very fast |
| **Why does it matter?** | Because the cost of reaching data drives nearly all performance decisions |
| **Where does it belong?** | The bottom layer, underneath every runtime and framework |
| **When should I use it?** | Whenever something is slow and the code itself looks correct |

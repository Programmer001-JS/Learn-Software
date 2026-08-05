# RAM

> **In one line —** the computer's working memory: fast, directly addressable, and completely erased the moment the power goes off.

| | |
|---|---|
| **Full name** | Random Access Memory |
| **Category** | Hardware Component |
| **Architectural Layer** | Physical / Memory |
| **Related notes** | [CPU](CPU.md) · [Cache Memory](Cache%20Memory.md) · [SSD and HDD](SSD%20and%20HDD.md) · [Memory Management](Memory%20Management.md) · [Process](Process.md) · [Redis](../08%20-%20Databases%20and%20Data/Redis.md) |

---

## 1. Short Definition

*What is it?*

RAM is the memory a running program actually works in. "Random access" means any address can be read in the same amount of time, unlike a spinning disk where position matters. It is **volatile** — its contents vanish when power is lost.

---

## 2. Purpose

*What is its main purpose?*

To hold everything a running program needs *right now*: its code, its variables, its open files' buffers. RAM exists because the [CPU](CPU.md) is far too fast to be fed directly from a disk.

---

## 3. Problem

*What engineering problem does it solve?*

Storage is permanent but slow; the CPU is fast but has almost no space. RAM is the compromise in the middle.

```text
CPU        ~0.3 ns   but only a few KB of registers
RAM        ~100 ns   gigabytes
SSD        ~100 µs   terabytes
```

Without RAM, every variable access would touch the disk and a program would run roughly a thousand times slower.

---

## 4. Architecture Position

RAM sits between the CPU caches and permanent storage.

```text
CPU
    ↓
Registers   (~1 KB,   ~0.3 ns)
    ↓
L1/L2/L3 Cache (~MB,  ~1–40 ns)
    ↓
RAM         (GB,      ~100 ns)     ← you are here
    ↓
SSD / HDD   (TB,      ~100 µs – 10 ms)
```

---

## 5. Real World Example

- **[Redis](../08%20-%20Databases%20and%20Data/Redis.md)** is built entirely on this property — it keeps the whole dataset in RAM to answer in microseconds.
- **PostgreSQL** keeps a *buffer pool* in RAM so frequently used pages are not re-read from disk.
- **Browsers** keep the parsed [DOM](../05%20-%20Frontend%20Architecture/DOM%20and%20CSSOM.md) and JavaScript heap in RAM — which is why many open tabs consume gigabytes.
- **AI model serving** loads model weights into RAM (or GPU memory) once, because reloading per request would be impossibly slow.

---

## 6. Input

*What does it receive as input?*

Read and write requests from the CPU, each carrying a **memory address** and, for writes, a value. Programs never see physical addresses — they see [virtual addresses](Memory%20Management.md) that the OS and hardware translate.

---

## 7. Processing

*What happens inside it?*

RAM is a grid of cells, each holding a bit as a charge in a tiny capacitor. To read, the controller selects a row and column and senses the charge. Because the charge leaks, DRAM must be **refreshed** thousands of times per second — this is why it needs constant power and why it forgets everything when the power stops.

---

## 8. Output

*What does it return?*

The bytes stored at the requested address, delivered over the memory bus, typically in ~50–100 nanoseconds. Data is transferred in blocks (**cache lines**, usually 64 bytes), not single bytes.

---

## 9. Internal Idea

*How does it work internally?*

Think of an enormous array of numbered boxes. Give the controller a number and it hands you the contents of that box — the same speed for box 3 and box 3 billion. That uniformity is what "random access" means, and it is the property a spinning [hard disk](SSD%20and%20HDD.md) does not have.

---

## 10. Communication

- **[CPU](CPU.md)** — via the memory bus, through the [cache](Cache%20Memory.md)
- **[Operating system](02%20-%20Operating%20Systems.md)** — which allocates it to [processes](Process.md) and maps virtual to physical addresses
- **Disk** — when the OS swaps memory pages in and out
- **GPU** — has its own separate, faster VRAM

---

## 11. Dependencies

- Continuous **power** — this is the defining constraint
- A **memory controller**, today integrated into the CPU
- The **operating system** to allocate and protect it

---

## 12. Alternatives

```text
Registers / Cache   faster, but tiny and not directly controllable
    ↓
RAM                 the working memory
    ↓
SSD                 ~1000× slower, but permanent and much larger
    ↓
Network storage     effectively unlimited, far slower again
```

There is no real substitute at this level — the question is only how much of your working set fits.

---

## 13. When To Use

> [!TIP]
> Keep in RAM whatever is accessed repeatedly and can be rebuilt if lost. That single rule is the entire justification for caching.

---

## 14. When NOT To Use

> [!CAUTION]
> Never keep data in RAM that you cannot afford to lose. A restart, a crash or a deploy erases it completely.
>
> Also, do not assume RAM is free: loading a 5 GB file into memory on a 4 GB machine does not fail gracefully — it triggers swapping, and the application becomes unusably slow long before it crashes.

---

## 15. Advantages

- ~1,000× faster than an SSD
- Uniform access time regardless of address
- Directly addressable by the CPU
- Cheap enough today that "add more RAM" is often the simplest real fix

---

## 16. Disadvantages

- **Volatile** — everything is lost on power loss
- Far more expensive per gigabyte than disk
- Physically limited by what the machine can hold
- Running out leads to swapping, which is catastrophically slow, or to the OS killing your process

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | ~100 ns per access — fast versus disk, slow versus cache |
| **Memory** | This *is* memory; exhausting it is a hard failure |
| **CPU** | A CPU stalled on a RAM read wastes hundreds of cycles |
| **GPU** | GPUs use separate VRAM; copying between them is a common bottleneck |
| **Network** | Caching in RAM removes network round trips entirely |

---

## 18. Security Considerations

> [!CAUTION]
> Anything in RAM is readable by anyone with sufficient privileges on the machine — including through crash dumps and memory-scraping malware.

- **Secrets in memory** — passwords and keys live in RAM in plaintext while in use; minimise how long
- **Memory-safety bugs** — buffer overflows and use-after-free in C/C++ are a leading source of vulnerabilities
- **Cold boot attacks** — RAM contents persist for seconds after power off
- **Shared hosting** — the OS isolates processes, but hardware side channels have broken that assumption before

---

## 19. Mental Model

> [!NOTE]
> **RAM = your desk. The disk = the filing cabinet.**
>
> You keep what you are working on right now on the desk because reaching it is instant. The cabinet holds everything permanently but takes time to open. And when you leave for the day, the desk is wiped clean.

---

## 20. Mini Architecture Diagram

```text
Process
    ↓
Virtual memory  (per-process illusion of a private address space)
    ↓
Operating System  →  page table translation
    ↓
Physical RAM
    ↓
Swap on disk  (only when RAM runs out — very slow)
```

---

## 21. Complete Request Flow

```text
Program reads variable X
    ↓
CPU checks L1 → L2 → L3 cache
    ↓
MISS — go to RAM
    ↓
Virtual address translated to physical address
    ↓
Memory controller reads a 64-byte cache line
    ↓
Value returned to CPU and stored in cache
    ↓
Next read of X takes ~1 ns instead of ~100 ns
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> RAM is fast, volatile working memory — put there whatever is read often and can be rebuilt if lost.

---

## 23. Common Mistakes

- **Treating RAM as storage** — anything not written to disk is gone after a restart
- **Loading entire large files into memory** instead of streaming them
- **Ignoring memory leaks** — memory that is allocated but never released eventually kills the process
- **Assuming free RAM is wasted RAM** — the OS deliberately uses spare memory as disk cache, so "high memory usage" is often healthy
- **Forgetting that containers have memory limits** — exceeding them gets the container killed, not slowed

---

## 24. Open Source Technologies

- **Redis**, **Valkey**, **Memcached** — in-memory data stores
- **Apache Ignite**, **Hazelcast** — distributed in-memory grids
- **valgrind**, **heaptrack** — memory profiling tools

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Check how much RAM your current project uses under normal load, and what happens when you double the load.
- [ ] Find one place where you load an entire collection into memory that could be streamed or paginated instead.
- [ ] Explain in two sentences why Redis is fast and why that speed has a cost.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
CPU
    ↓
Cache
    ↓
RAM
    ↓
SSD / HDD
```

## 2. Request Flow

```text
Input       a memory address
    ↓
Processing  address translation, row/column selection, cache line read
    ↓
Output      64 bytes of data delivered to the CPU cache
```

## 3. Real-World Usage

**Redis** exists purely to exploit this layer: by keeping the entire working set in RAM it answers in microseconds, and it accepts volatility as the price. Twitter, GitHub and Stack Overflow all rely on that trade.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Fast, volatile working memory for running programs |
| **Why does it exist?** | Because the CPU is far too fast to be fed from a disk |
| **Where does it belong?** | Between the CPU caches and permanent storage |
| **When should I use it?** | For anything read repeatedly that can be rebuilt if lost |

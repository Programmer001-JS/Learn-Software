# Memory Management

> **In one line —** the kernel gives every process its own private, imaginary memory and quietly maps it onto the real thing.

| | |
|---|---|
| **Category** | Kernel Subsystem |
| **Architectural Layer** | Kernel Space |
| **Related notes** | [RAM](RAM.md) · [Kernel](Kernel.md) · [Process](Process.md) · [User Space](User%20Space.md) · [SSD and HDD](SSD%20and%20HDD.md) |

---

## 1. Short Definition

*What is it?*

Memory management is how the [kernel](Kernel.md) allocates [RAM](RAM.md) to processes, isolates them from each other, and maintains the illusion that each has a large, private, contiguous address space — regardless of what physical memory actually looks like.

---

## 2. Purpose

*What is its main purpose?*

Three things at once: **isolation** (one process cannot read another's memory), **abstraction** (every process sees a clean address space starting at zero), and **efficiency** (physical RAM is shared, reused and overcommitted).

---

## 3. Problem

*What engineering problem does it solve?*

Without it, programs would need to know which physical addresses were free, coordinate with each other, and could read anything. Physical memory is also fragmented and finite, while programs want large contiguous regions.

---

## 4. Architecture Position

```text
Process A                Process B
virtual addresses        virtual addresses
0x0000 → 0xFFFF          0x0000 → 0xFFFF        ← same numbers, different memory
    ↓                        ↓
┌───────── KERNEL: page tables + MMU ─────────┐
└──────────────────┬───────────────────────────┘
                   ↓
          Physical RAM (shared, fragmented)
                   ↓
              Swap on disk  (when RAM runs out)
```

---

## 5. Real World Example

- **Container OOM kills** — exit code 137 means the kernel's out-of-memory killer chose your process. It is a memory-management decision, not an application crash.
- **Copy-on-write** — `fork()` creates a child process instantly by sharing all pages read-only and copying only what is written. Gunicorn and Celery pre-fork workers rely on this.
- **Memory-mapped files** — how databases and `mmap`-based tools read huge files without loading them into RAM.

---

## 6. Input

*What does it receive as input?*

Allocation requests (`malloc`, `mmap`, or your language's allocator underneath), memory accesses from running code, and **page faults** raised by the hardware when a virtual address has no physical page behind it yet.

---

## 7. Processing

*What happens inside it?*

Memory is handled in fixed-size **pages**, typically 4 KB. The kernel maintains a **page table** per process mapping virtual pages to physical frames, and the CPU's **MMU** performs that translation on every single access, in hardware.

```text
Process reads address 0x7F3A0010
    ↓
MMU looks up the page table  (cached in the TLB)
    ↓
Mapped?  → YES → access physical RAM
    ↓ NO
PAGE FAULT → kernel takes over
    ↓
Allocate a page / load it from disk / kill the process if invalid
```

---

## 8. Output

*What does it return?*

Usable memory for the process — or a hard failure: an allocation error, a segmentation fault for an invalid access, or termination by the OOM killer when the machine is out of memory.

---

## 9. Internal Idea

*How does it work internally?*

Two ideas do most of the work.

**Virtual memory** — every process has its own map, so identical addresses in two processes point at different physical pages. Isolation is not a check that can be bypassed; the mapping simply does not exist.

**Lazy allocation** — asking for memory does not consume it. The kernel promises the address range and only allocates physical pages when they are actually touched.

> [!IMPORTANT]
> This is why Linux **overcommits**: it hands out more memory than it has, betting that not everyone will use their full allocation. The bet is usually right — and when it is wrong, the OOM killer resolves it.

---

## 10. Communication

- **[CPU](CPU.md) / MMU** — hardware performs the translation
- **Processes** — through allocation calls and page faults
- **[Disk](SSD%20and%20HDD.md)** — for swap and memory-mapped files
- **cgroups** — how containers get memory limits

---

## 11. Dependencies

- An **MMU** in the CPU
- Page tables maintained by the kernel
- Swap space, if the machine is configured with it

---

## 12. Alternatives

```text
Virtual memory + paging     every modern general-purpose OS
    ↓
Segmentation                older, largely historical
    ↓
No virtual memory           small embedded systems; fast, no isolation
```

Within a process, the choice is between manual management (C, C++, Rust), garbage collection (Java, Go, Python, JavaScript) and ownership-based automatic freeing (Rust).

---

## 13. When To Use

> [!TIP]
> You do not manage kernel memory, but you must think about it whenever you set container memory limits, size a cache, or decide whether to stream a file rather than load it.

---

## 14. When NOT To Use

> [!CAUTION]
> Do not rely on swap to cover insufficient RAM. Swapping is roughly a thousand times slower than RAM; an application that starts swapping is effectively down while still appearing alive — often worse than an outright crash.

---

## 15. Advantages

- Enforced isolation between processes
- Each process sees a clean, contiguous address space
- More memory can be promised than physically exists
- Shared libraries and forked processes share pages instead of duplicating them

---

## 16. Disadvantages

- Address translation costs time on every access (mitigated by the TLB)
- Page faults are expensive; swapping is catastrophic
- Overcommitment means allocation can succeed and *usage* can still fail later
- Fragmentation and memory leaks are still entirely possible inside a process

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Translation is nearly free when the TLB hits; a page fault costs microseconds to milliseconds |
| **Memory** | Page tables themselves consume memory |
| **CPU** | Garbage collection pauses and page-fault handling both consume CPU |
| **GPU** | VRAM is managed separately and is usually far scarcer |
| **Network** | Not directly involved |

---

## 18. Security Considerations

> [!CAUTION]
> Memory-safety bugs — buffer overflows, use-after-free, double free — remain one of the largest single sources of serious vulnerabilities in software. This is the entire reason Rust exists and why memory-safe languages are increasingly mandated for new systems code.

- **ASLR** randomises addresses so attackers cannot predict where to jump
- **DEP/NX** marks data pages non-executable, so injected data cannot be run
- **Secrets in memory** live in plaintext; minimise their lifetime and avoid writing them to core dumps or logs
- **Spectre/Meltdown** read across process boundaries without any software bug at all

---

## 19. Mental Model

> [!NOTE]
> **Virtual memory is a hotel with room numbers that mean nothing outside your own booking.**
>
> Every guest is told they are in "room 101". The front desk knows which physical room that really is, and no guest can find another's. If the hotel is full, the manager moves someone's luggage into the basement (swap) — and getting it back takes a very long time.

---

## 20. Mini Architecture Diagram

```text
Process virtual address space
    ↓
Page table  (per process)
    ↓
MMU  (hardware translation, TLB-cached)
    ↓
Physical RAM
    ↓
Swap  (disk — the emergency exit you do not want to use)
```

---

## 21. Complete Request Flow

Allocating and using memory:

```text
Program calls malloc(1 GB)
    ↓
Kernel reserves the ADDRESS RANGE — no physical RAM used yet
    ↓
Program writes to the first byte
    ↓
PAGE FAULT — no physical page mapped
    ↓
Kernel allocates one 4 KB page, updates the page table
    ↓
Write succeeds
    ↓
...repeated per page as the program actually touches memory
    ↓
If physical RAM runs out:
        swap pages to disk (very slow)  or  OOM killer terminates a process
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> Every process gets a private virtual address space that the kernel maps onto shared physical RAM — allocation is a promise, and the promise is only cashed when memory is actually touched.

---

## 23. Common Mistakes

- **Assuming allocation means the memory exists** — overcommitment means failure comes later
- **Not setting container memory limits**, or setting them without understanding OOM behaviour
- **Relying on swap** instead of provisioning enough RAM
- **Loading whole files or result sets into memory** instead of streaming or paginating
- **Reading "high memory usage" as a problem** — the kernel deliberately uses free RAM as a page cache
- **Ignoring memory leaks** in long-running processes, where they are eventually fatal

---

## 24. Open Source Technologies

- **jemalloc**, **tcmalloc** — alternative allocators used by databases and high-throughput servers
- **valgrind**, **heaptrack**, **AddressSanitizer** — find leaks and memory errors
- **cgroups v2** — container memory limits
- **Rust** — memory safety enforced at compile time, without garbage collection

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Watch your application's memory over an hour under load. Does it stabilise or keep climbing?
- [ ] Set a memory limit on a container and deliberately exceed it. Observe exactly how it fails.
- [ ] Find one place where you load a whole collection into memory and convert it to streaming or pagination.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Process (virtual addresses)
    ↓
Page table + MMU
    ↓
Physical RAM
    ↓
Swap (disk)
```

## 2. Request Flow

```text
Input       an allocation request or a memory access
    ↓
Processing  virtual→physical translation; page fault handling; eviction under pressure
    ↓
Output      usable memory, or a fault, or process termination
```

## 3. Real-World Usage

**Redis** relies on `fork()` and copy-on-write to take a background snapshot: the child process shares all pages with the parent until they are modified. A kernel memory-management feature is what makes persistence possible without pausing the server.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The kernel's allocation, isolation and mapping of memory |
| **Why does it exist?** | For isolation, a clean abstraction, and efficient sharing of scarce RAM |
| **Where does it belong?** | Between processes and physical RAM, with hardware assistance |
| **When should I use it?** | Think about it whenever setting limits, sizing caches or handling large data |

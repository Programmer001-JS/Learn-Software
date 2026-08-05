# Cache Memory

> **In one line —** a small, very fast memory sitting inside the CPU that keeps recently used data close, so the processor does not have to wait for RAM.

| | |
|---|---|
| **Category** | Hardware Component |
| **Architectural Layer** | Physical / Memory |
| **Levels** | L1, L2, L3 |
| **Related notes** | [CPU](CPU.md) · [RAM](RAM.md) · [Registers](Registers.md) · [Cache](../08%20-%20Databases%20and%20Data/Cache.md) · [Caching](../10%20-%20Distributed%20Systems/Caching.md) |

---

## 1. Short Definition

*What is it?*

CPU cache is a small block of extremely fast memory built into the processor. It automatically keeps copies of recently and frequently used data from [RAM](RAM.md), so that the next access takes about a nanosecond instead of a hundred.

---

## 2. Purpose

*What is its main purpose?*

To hide the enormous speed gap between the [CPU](CPU.md) and RAM. Without it, a modern processor would spend most of its life stalled, waiting for data to arrive.

---

## 3. Problem

*What engineering problem does it solve?*

CPUs got dramatically faster over the decades; RAM did not keep up.

```text
CPU cycle     ~0.3 ns
RAM access    ~100 ns      ← roughly 300 wasted cycles per access
```

A processor that fetched every value straight from RAM would run at a small fraction of its rated speed. Cache closes that gap for the data that is actually being used.

---

## 4. Architecture Position

Cache sits **between the registers and RAM**, physically inside the CPU package.

```text
CPU core
    ↓
Registers      ~1 KB          ~0.3 ns
    ↓
L1 Cache       ~32–64 KB      ~1 ns      per core
    ↓
L2 Cache       ~256 KB–2 MB   ~4 ns      per core
    ↓
L3 Cache       ~8–64 MB       ~40 ns     shared by all cores
    ↓
RAM            GB             ~100 ns
```

---

## 5. Real World Example

- **Game engines** organise entities in contiguous arrays instead of scattered objects (data-oriented design) purely to keep the cache hit rate high — often a 2–10× speedup with identical logic.
- **Databases** store rows in fixed-size pages so a single cache line fetch brings in useful neighbouring data.
- **Numerical libraries** (NumPy, BLAS) get much of their speed from cache-aware loop ordering, not from cleverer maths.

---

## 6. Input

*What does it receive as input?*

A memory address requested by the CPU. The cache checks whether it already holds the block containing that address.

---

## 7. Processing

*What happens inside it?*

```text
CPU requests address X
    ↓
Is it in L1?  → YES → return in ~1 ns          (cache HIT)
    ↓ NO
Is it in L2?  → YES → return in ~4 ns
    ↓ NO
Is it in L3?  → YES → return in ~40 ns
    ↓ NO
Fetch from RAM (~100 ns)                        (cache MISS)
    ↓
Store the 64-byte cache line in L1/L2/L3
    ↓
Evict something else if full  (usually least recently used)
```

Crucially, the cache never fetches a single byte — it fetches a whole **cache line**, typically 64 bytes. Reading one array element brings its neighbours along for free.

---

## 8. Output

*What does it return?*

The requested bytes, plus a side effect: the surrounding cache line is now resident, so nearby addresses will be fast for a while.

---

## 9. Internal Idea

*How does it work internally?*

Two assumptions about how programs behave, both usually true:

- **Temporal locality** — data used now will probably be used again soon
- **Spatial locality** — data next to what you used will probably be used soon

> [!IMPORTANT]
> This is exactly the same idea as [Redis](../08%20-%20Databases%20and%20Data/Redis.md) in front of a database, or a CDN in front of a web server. **Caching is one concept that appears at every layer of computing**, from 64 bytes inside a chip to gigabytes across continents.

---

## 10. Communication

- **[Registers](Registers.md)** and the CPU core above it
- **[RAM](RAM.md)** below it
- **Other cores** — L3 is shared, and cores must keep their caches coherent with each other

---

## 11. Dependencies

- Built into the CPU; not separately installable or configurable
- Depends on **program access patterns** — the software decides, indirectly, how effective it is

---

## 12. Alternatives

There is no alternative at this level — you cannot switch it off or replace it. What you *can* change is how well your code uses it:

```text
Array of structs, iterated in order     →  excellent cache behaviour
Linked list of scattered objects        →  a cache miss per element
Random access into a huge hash map      →  mostly misses
```

---

## 13. When To Use

> [!TIP]
> You never "use" cache directly — you write code that is friendly to it: iterate arrays in order, keep hot data small and contiguous, avoid pointer-chasing in tight loops.

---

## 14. When NOT To Use

> [!CAUTION]
> Do not optimise for cache behaviour in ordinary application code. In a web backend, a single database query costs more than millions of cache misses. This matters in tight numeric loops, game engines and database internals — not in your CRUD endpoint.

---

## 15. Advantages

- Roughly 100× faster than RAM
- Completely automatic — no code required
- Makes sequential access patterns dramatically cheaper
- Shared L3 helps multi-core workloads on common data

---

## 16. Disadvantages

- Very small — measured in KB and MB, not GB
- Not directly controllable by the programmer
- **Cache coherence** between cores adds cost in multithreaded code
- **False sharing** — two threads writing to adjacent variables on the same cache line silently destroy each other's performance

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | The difference between a hit and a miss is ~100× |
| **Memory** | Tiny footprint, but it shapes how you should lay out data |
| **CPU** | A miss stalls the core for hundreds of cycles |
| **GPU** | GPUs have their own, smaller caches and rely on bandwidth instead |
| **Network** | Not involved |

---

## 18. Security Considerations

> [!CAUTION]
> **Spectre** and **Meltdown** are cache-timing attacks: by measuring whether an access was fast or slow, an attacker can infer memory contents they were never allowed to read. Cache is one of the few hardware features with a serious security history.

Practical consequence for application developers: use **constant-time comparison** for secrets, tokens and password hashes, so that timing reveals nothing.

---

## 19. Mental Model

> [!NOTE]
> **Cache = the desk you are working at. RAM = the shelf across the room.**
>
> Whatever is on the desk is instantly at hand. Anything else means standing up. And when you fetch one folder from the shelf, you bring the whole stack around it, because you will probably need those too.

---

## 20. Mini Architecture Diagram

```text
       ┌──────── CPU ────────┐
       │  Core 1     Core 2  │
       │   ↓  L1      ↓  L1  │
       │   ↓  L2      ↓  L2  │
       │   └──── L3 ─────┘   │   (shared)
       └──────────┬──────────┘
                  ↓
                 RAM
```

---

## 21. Complete Request Flow

**Cache hit:**

```text
CPU needs value X
    ↓
Found in L1
    ↓
Returned in ~1 ns — no stall
```

**Cache miss:**

```text
CPU needs value X
    ↓
L1 miss → L2 miss → L3 miss
    ↓
Read 64-byte cache line from RAM  (~100 ns, core stalls)
    ↓
Line stored in L1/L2/L3, something else evicted
    ↓
Value delivered — and X's neighbours are now fast too
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> Cache exists because RAM cannot keep up with the CPU; code that accesses memory in predictable, contiguous patterns can be several times faster than identical logic that jumps around.

---

## 23. Common Mistakes

- **Confusing CPU cache with application cache** — same idea, wildly different scale
- **Micro-optimising cache behaviour in I/O-bound code**, where it is irrelevant
- **Using linked structures in hot numeric loops** and wondering why it is slow
- **False sharing in multithreaded code** — padding shared counters is a real fix
- **Assuming you can control it** — you can only influence it through data layout

---

## 24. Open Source Technologies

- **perf**, **cachegrind** (valgrind), **Intel VTune** — measure real cache hit rates
- **NumPy**, **OpenBLAS**, **Eigen** — libraries built around cache-aware algorithms

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Write two loops over the same 2D array, one row-major and one column-major, and time them. Explain the difference.
- [ ] List every layer of caching in a typical web application, from CPU cache to CDN.
- [ ] Explain temporal and spatial locality in your own words, with an example from daily life.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Registers
    ↓
L1 → L2 → L3 Cache
    ↓
RAM
    ↓
Disk
```

## 2. Request Flow

```text
Input       a memory address
    ↓
Processing  check L1 → L2 → L3; on miss, fetch a 64-byte line from RAM
    ↓
Output      the value, plus its neighbours now cached
```

## 3. Real-World Usage

**Game engines** such as Unreal restructure their data into flat contiguous arrays specifically to keep the CPU cache hot. The gameplay logic is unchanged; the memory layout alone accounts for the frame-rate difference.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Small, very fast memory inside the CPU holding recently used data |
| **Why does it exist?** | Because RAM is ~100× too slow for a modern processor |
| **Where does it belong?** | Between the registers and RAM |
| **When should I use it?** | Indirectly, by writing code with predictable, contiguous memory access |

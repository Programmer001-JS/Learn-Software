# SSD and HDD

> **In one line —** permanent storage that survives power loss; the SSD has no moving parts and is ~100× faster, the HDD is a spinning magnetic disk and is ~10× cheaper per terabyte.

| | |
|---|---|
| **Full names** | Solid State Drive · Hard Disk Drive |
| **Category** | Hardware Component |
| **Architectural Layer** | Physical / Storage |
| **Related notes** | [RAM](RAM.md) · [File Systems](File%20Systems.md) · [Database Fundamentals](../08%20-%20Databases%20and%20Data/Database%20Fundamentals.md) · [Backup Strategy](../14%20-%20Scalability%20and%20Reliability/Backup%20Strategy.md) |

---

## 1. Short Definition

*What is it?*

Both are **persistent storage** — they keep data when the power is off. An **HDD** stores bits magnetically on spinning platters read by a moving arm. An **SSD** stores bits as electrical charge in flash memory chips, with no moving parts at all.

---

## 2. Purpose

*What is its main purpose?*

To hold everything that must survive a restart: the operating system, your code, databases, user uploads, logs, backups. [RAM](RAM.md) is fast but forgets; storage remembers but is slow.

---

## 3. Problem

*What engineering problem does it solve?*

RAM loses everything on power loss and is far too expensive to hold terabytes. Storage solves permanence and capacity, and pays for it in latency.

```text
RAM     ~100 ns      volatile     expensive per GB
SSD     ~100 µs      permanent    moderate            (~1,000× slower than RAM)
HDD     ~10 ms       permanent    cheap               (~100,000× slower than RAM)
```

---

## 4. Architecture Position

Storage sits below RAM, at the bottom of the local memory hierarchy.

```text
CPU → Cache → RAM
                ↓
        SSD / HDD          ← permanent, slow, large
                ↓
    Object storage / network storage  (S3, MinIO)
                ↓
        Tape / cold archive
```

---

## 5. Real World Example

- **Databases** live on SSDs. PostgreSQL's entire design — indexes, write-ahead logs, buffer pools — is shaped by the cost of a disk read.
- **Netflix** stores video on cheap high-capacity drives at the edge, because streaming is sequential and does not need SSD random-access speed.
- **Backups and archives** still use HDDs and even tape, because cost per terabyte dominates when access is rare.
- **Docker images and CI pipelines** are heavily disk-bound; this is why build servers use SSDs.

---

## 6. Input

*What does it receive as input?*

Read and write requests from the [operating system](02%20-%20Operating%20Systems.md), addressed in **blocks** (typically 4 KB), not individual bytes. Applications never talk to a disk directly — they go through the [file system](File%20Systems.md) and [system calls](System%20Calls.md).

---

## 7. Processing

*What happens inside it?*

**HDD:**
```text
Request block N
    ↓
Move the arm to the right track      ← "seek", ~5–10 ms, mechanical
    ↓
Wait for the platter to rotate       ← "rotational latency", ~4 ms
    ↓
Read the block magnetically
```

**SSD:**
```text
Request block N
    ↓
Controller maps the logical block to a physical flash page
    ↓
Read the charge from the flash cells  ← ~100 µs, purely electronic
```

> [!IMPORTANT]
> The HDD's cost is **mechanical movement**. This is why random access on an HDD is catastrophic while sequential reading is respectable — and why the SSD, which has no arm to move, made random access affordable and changed database design.

---

## 8. Output

*What does it return?*

The requested blocks of data. The OS caches them in RAM (the *page cache*), so a repeated read of the same file usually never reaches the disk at all.

---

## 9. Internal Idea

*How does it work internally?*

- **HDD** — a record player: a spinning magnetic platter and an arm that must physically travel to the right place.
- **SSD** — a grid of flash cells holding charge, with a controller that spreads writes evenly across the chips (**wear levelling**), because each cell can only be rewritten a limited number of times.

That last detail matters: SSDs cannot overwrite in place. They must erase whole blocks first, which is why sustained heavy writing is slower than a benchmark suggests and why SSDs eventually wear out.

---

## 10. Communication

- **[Operating system](02%20-%20Operating%20Systems.md)** — through the file system and block layer
- **[RAM](RAM.md)** — via the OS page cache, and via swap when memory runs out
- **Interfaces** — SATA (older, ~550 MB/s) or NVMe over PCIe (modern SSDs, several GB/s)

---

## 11. Dependencies

- A [file system](File%20Systems.md) to give structure to raw blocks
- Operating system drivers
- For SSDs, a controller that handles wear levelling and TRIM

---

## 12. Alternatives

```text
RAM disk            fastest, volatile, tiny
    ↓
NVMe SSD            very fast random access, moderate cost
    ↓
SATA SSD            fast, cheaper
    ↓
HDD                 cheap per TB, terrible random access
    ↓
Object storage      S3 / MinIO — network-attached, unlimited, higher latency
    ↓
Tape / cold archive almost free, retrieval measured in hours
```

---

## 13. When To Use

> [!TIP]
> **SSD** for anything with random access or latency requirements — databases, application servers, build machines.
> **HDD** for large sequential data where cost dominates — backups, archives, video libraries, logs.

---

## 14. When NOT To Use

> [!CAUTION]
> - Do not run a database on an HDD if it does random reads; the difference is not 2×, it is closer to 100×.
> - Do not store user uploads on the local disk of an application server — the moment you add a second server, half the requests will not find the file. Use object storage instead.
> - Do not treat a single disk as a backup. RAID is not a backup either; it protects against hardware failure, not against deletion.

---

## 15. Advantages

**SSD**
- ~100× faster random access than HDD
- No moving parts, so more durable and silent
- Low latency makes databases and boot times dramatically better

**HDD**
- Much cheaper per terabyte
- Very large capacities
- Predictable, well-understood failure behaviour

---

## 16. Disadvantages

**SSD**
- More expensive per terabyte
- Limited write endurance; cells wear out
- Recovery after failure is often harder than on an HDD

**HDD**
- Mechanically fragile — moving parts fail
- Random access is brutally slow
- Higher power draw and heat in large arrays

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | The dominant factor in most database workloads |
| **Memory** | The OS uses spare RAM as a page cache to avoid touching the disk |
| **CPU** | Low, but a process blocked on I/O still occupies a thread unless you use async I/O |
| **GPU** | Only relevant when a slow disk starves the data pipeline during training |
| **Network** | Network storage adds latency on top of disk latency |

---

## 18. Security Considerations

> [!CAUTION]
> **Deleting a file does not erase the data** — it removes the pointer. On an SSD, wear levelling means old copies can persist in cells the file system no longer references, so overwriting is not reliable either.

- **Encryption at rest** (LUKS, BitLocker, cloud-managed keys) is the only dependable protection for a lost or decommissioned drive
- **Physical destruction** is the standard for disposing of drives that held sensitive data
- Backups inherit the sensitivity of what they contain, and are frequently the least protected copy

---

## 19. Mental Model

> [!NOTE]
> **HDD = a record player.** An arm physically travels to the right groove; jumping around the record is slow.
>
> **SSD = a book with an index.** Any page can be opened directly, at the same cost.
>
> **RAM = your desk.** Instant, but cleared every night.

---

## 20. Mini Architecture Diagram

```text
Application
    ↓  read() / write()
Operating System
    ↓
Page cache in RAM   ← most reads stop here
    ↓  (miss)
File system
    ↓
Block layer / driver
    ↓
SSD or HDD
```

---

## 21. Complete Request Flow

```text
Application opens and reads a file
    ↓
System call into the kernel
    ↓
Is the block already in the page cache (RAM)?
    ↓ YES → return immediately (~100 ns)
    ↓ NO
File system translates the path to block numbers
    ↓
Disk reads the blocks   (SSD ~100 µs / HDD ~10 ms)
    ↓
Blocks cached in RAM for next time
    ↓
Data returned to the application
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> Storage is permanent and slow; the entire design of databases and file systems is an effort to touch it as rarely as possible.

---

## 23. Common Mistakes

- **Storing user uploads on a local application-server disk** instead of object storage
- **Running a database on an HDD** and blaming the query planner
- **Confusing RAID with backup** — RAID survives a dead disk, not a `DELETE`
- **Assuming deleted means erased**
- **Ignoring disk full** — a full disk breaks databases and logging in confusing ways
- **Not monitoring free space** — one of the most common and most avoidable production incidents

---

## 24. Open Source Technologies

- **ext4**, **XFS**, **Btrfs**, **ZFS** — Linux file systems
- **MinIO** — self-hosted S3-compatible object storage
- **Ceph** — distributed storage
- **smartmontools**, **fio**, **iostat** — health monitoring and benchmarking

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Find out what kind of disk your database currently runs on, and what its latency is.
- [ ] Identify anything in your project stored on a local disk that would break if you added a second server.
- [ ] Write down your backup plan in three sentences: what, how often, and how you would restore it.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
CPU → Cache → RAM
                ↓
          SSD / HDD
                ↓
       Object storage (S3, MinIO)
```

## 2. Request Flow

```text
Input       a read or write request for a block
    ↓
Processing  page cache check → file system → block layer → physical media
    ↓
Output      data returned and cached in RAM
```

## 3. Real-World Usage

**PostgreSQL** was designed around the assumption that disk access is expensive: indexes exist to avoid full scans, the buffer pool exists to avoid re-reading, and the write-ahead log exists to make writes sequential. The whole architecture is a response to this one hardware property.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Persistent storage — SSD via flash chips, HDD via spinning magnetic platters |
| **Why does it exist?** | Because RAM forgets everything on power loss and cannot hold terabytes affordably |
| **Where does it belong?** | Below RAM, at the bottom of the local memory hierarchy |
| **When should I use it?** | SSD for random access and databases; HDD for cheap sequential bulk storage |

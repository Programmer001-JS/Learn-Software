# User Space

> **In one line —** the restricted world where all your programs run, unable to touch hardware, and therefore unable to destroy the machine.

| | |
|---|---|
| **Category** | System Software Concept |
| **Architectural Layer** | User Space |
| **Related notes** | [Kernel](Kernel.md) · [System Calls](System%20Calls.md) · [Process](Process.md) · [Memory Management](Memory%20Management.md) · [Operating Systems](02%20-%20Operating%20Systems.md) |

---

## 1. Short Definition

*What is it?*

User space is the unprivileged CPU mode in which all ordinary programs run — your API, your database, your browser, your shell. Code there cannot access hardware, cannot read another process's memory, and cannot execute privileged instructions.

---

## 2. Purpose

*What is its main purpose?*

To contain damage. A crash, a bug or an attack in user space affects **one process**, not the machine. Everything else in the system depends on this containment being unbreakable.

---

## 3. Problem

*What engineering problem does it solve?*

Early computers ran everything with full privileges, so any bug could overwrite the operating system. Separating a privileged [kernel](Kernel.md) from unprivileged user code turned "programs must behave" into "programs *cannot* misbehave", enforced by the CPU.

---

## 4. Architecture Position

```text
┌──────────── USER SPACE ─────────────┐
│  your API      PostgreSQL           │  ← unprivileged
│  browser       shell      Docker    │     isolated
│                                      │     crashes are contained
└──────────────────┬───────────────────┘
                   │  system calls — the only way through
┌──────────────────▼──── KERNEL SPACE ─┐
│  full hardware access                │
└──────────────────────────────────────┘
```

> [!IMPORTANT]
> Almost everything you will ever write runs in user space. Even a database, even Docker, even Kubernetes — they are all ordinary user-space processes that ask the kernel for things.

---

## 5. Real World Example

- **PostgreSQL** is a user-space process. Its speed comes from asking the kernel for the right things (sequential reads, batched writes), not from privileged access.
- **Docker daemon** runs in user space and configures kernel features on your behalf.
- **A segfault** in your program kills that process only; the machine and every other process continue.

---

## 6. Input

*What does it receive as input?*

CPU time granted by the [scheduler](Scheduler.md), memory granted by the [memory manager](Memory%20Management.md), and data returned from [system calls](System%20Calls.md). A user-space process receives nothing directly from hardware.

---

## 7. Processing

*What happens inside it?*

Ordinary computation: arithmetic, branching, working with its own memory. All of that is fast and requires no kernel involvement.

The moment the program needs something real — a file, a socket, more memory, a new process — it must make a system call and briefly hand control to the kernel.

```text
User space work        no kernel involved, full speed
    ↓
Needs a file / socket / memory
    ↓
SYSTEM CALL  →  kernel mode  →  back to user mode
    ↓
User space work continues
```

---

## 8. Output

*What does it return?*

Whatever the program produces: an HTTP response, a written file, a rendered frame — always achieved by asking the kernel to perform the final privileged step.

---

## 9. Internal Idea

*How does it work internally?*

The CPU has privilege levels. Kernel code runs at ring 0 with every instruction available; user code runs at ring 3, where privileged instructions simply fault. Each process also gets its **own virtual address space**, so pointer `0x1000` in one process and in another refer to entirely different physical memory.

> [!IMPORTANT]
> This is why a process cannot read another's memory even if it tries: the addresses do not resolve to the same physical pages. It is not a check that can be bypassed — the mapping simply does not exist.

---

## 10. Communication

- **[Kernel](Kernel.md)** — via [system calls](System%20Calls.md) and signals
- **Other processes** — only through kernel-mediated channels: sockets, pipes, shared memory, files
- **Hardware** — never directly

---

## 11. Dependencies

- A [kernel](Kernel.md) to serve its requests
- CPU privilege levels and an MMU to enforce isolation
- Standard libraries (libc, the Go runtime, the JVM) that wrap system calls in something usable

---

## 12. Alternatives

```text
User space          normal, isolated, safe                  ← where you belong
    ↓
Kernel modules      full power, no safety net
    ↓
eBPF                run sandboxed, verified code in the kernel — the modern middle ground
    ↓
Unikernel           one application compiled together with a minimal kernel
```

---

## 13. When To Use

> [!TIP]
> Always. If you can solve a problem in user space, solve it there. Moving code into the kernel trades a contained crash for a machine-wide one.

---

## 14. When NOT To Use

> [!CAUTION]
> User space is unsuitable only when you genuinely need privileged hardware access — a device driver, a file system implementation, or extremely high-throughput packet processing. Even then, prefer **eBPF** or user-space frameworks (FUSE, DPDK) over writing kernel code.

---

## 15. Advantages

- Crashes are contained to one process
- Processes cannot read each other's memory
- Debuggers, profilers and normal tooling all work
- Any language, any runtime, no special privileges

---

## 16. Disadvantages

- Every real operation costs a system call and a mode switch
- No direct hardware access, so some latency floors cannot be beaten
- Copying data between user and kernel buffers is pure overhead in I/O-heavy systems

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Pure computation runs at full speed; each system call adds ~100 ns – 1 µs |
| **Memory** | Each process has its own address space and page tables |
| **CPU** | Mode switches and context switches are overhead |
| **GPU** | Access goes through kernel drivers |
| **Network** | High-throughput systems reduce crossings with `io_uring`, `sendfile` or batching |

---

## 18. Security Considerations

> [!CAUTION]
> User space isolation is strong but not absolute. **Privilege escalation** bugs let user code reach the kernel, and hardware side channels (Spectre, Meltdown) have read across the boundary without any software bug at all.

- Run processes as **non-root** — root in user space is still user space, but a compromise reaches much further
- Use **seccomp** to restrict which system calls a process may make
- Containers are user-space isolation plus kernel namespaces — good, but weaker than a virtual machine

---

## 19. Mental Model

> [!NOTE]
> **User space = a tenant's flat. Kernel space = the building's utility room.**
>
> You can do whatever you like inside your own flat. You cannot enter the utility room, and you cannot walk into your neighbour's flat. Anything involving water or electricity means asking the manager.

---

## 20. Mini Architecture Diagram

```text
┌── USER SPACE ────────────────────────────┐
│  Process A        Process B              │
│  own memory       own memory             │
│      │                │                  │
└──────┼────────────────┼──────────────────┘
       │  syscall       │  syscall
┌──────▼────────────────▼─── KERNEL ───────┐
│  scheduler · memory · drivers · network   │
└───────────────────────────────────────────┘
```

---

## 21. Complete Request Flow

An HTTP request arriving at your API:

```text
Packet arrives at the network card
    ↓
Kernel (interrupt → network stack → socket buffer)
    ↓
Your process is waiting in accept()/read()
    ↓
CROSS INTO USER SPACE — data copied to your buffer
    ↓
Your code runs: routing, validation, business logic   ← no kernel involvement
    ↓
Needs the database → SYSTEM CALL to write to a socket
    ↓
Kernel sends the packet
    ↓
Response written back the same way
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> Your code runs in a sandbox with no hardware access; every real effect it has on the world is a request the kernel performs on its behalf.

---

## 23. Common Mistakes

- **Not knowing where the boundary is**, and therefore not understanding why I/O is expensive
- **Making many tiny system calls** instead of buffering — a classic cause of slow file and socket code
- **Running containers as root** unnecessarily
- **Assuming container isolation equals virtual machine isolation**
- **Blaming the language** for I/O latency that is really mode-switch and copy overhead

---

## 24. Open Source Technologies

- **glibc**, **musl** — the C libraries that wrap system calls
- **seccomp**, **AppArmor**, **SELinux** — restricting what user space may ask for
- **FUSE** — file systems implemented in user space
- **io_uring** — modern Linux interface that dramatically reduces syscall overhead
- **strace**, **ltrace** — observe the boundary being crossed

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Write a program that reads a file byte by byte, then one that reads it in 64 KB blocks. Time both and explain the difference.
- [ ] Check whether your containers run as root, and change one that does not need to.
- [ ] Draw the user/kernel boundary and mark every crossing in one HTTP request.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Your application
    ↓
Standard library
    ↓
System calls  ← the boundary
    ↓
Kernel
    ↓
Hardware
```

## 2. Request Flow

```text
Input       CPU time and memory granted by the kernel
    ↓
Processing  unprivileged computation; system calls for anything real
    ↓
Output      results, with all hardware effects performed by the kernel
```

## 3. Real-World Usage

**PostgreSQL, Nginx and Redis** are all user-space processes. Their performance engineering is largely about minimising how often they cross into the kernel and how much data is copied when they do.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The unprivileged, isolated mode where all normal programs run |
| **Why does it exist?** | So that one program's failure cannot damage the machine |
| **Where does it belong?** | Above the kernel, below nothing |
| **When should I use it?** | Always — leave it only for genuine driver-level work |

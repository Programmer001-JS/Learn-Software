# Kernel

> **In one line —** the core of the operating system: the only code allowed to touch hardware directly, and the referee between every program on the machine.

| | |
|---|---|
| **Category** | System Software |
| **Architectural Layer** | Kernel Space |
| **Related notes** | [Operating Systems](02%20-%20Operating%20Systems.md) · [User Space](User%20Space.md) · [System Calls](System%20Calls.md) · [Scheduler](Scheduler.md) · [Memory Management](Memory%20Management.md) · [Docker](../13%20-%20DevOps%20and%20Delivery/Docker.md) |

---

## 1. Short Definition

*What is it?*

The kernel is the central part of an operating system. It runs in a privileged CPU mode with **complete access to hardware**, manages memory and processes, and decides what every program is allowed to do.

---

## 2. Purpose

*What is its main purpose?*

To be the single trusted authority over the machine. Because only the kernel can touch hardware, every program must go through it — which is exactly what makes isolation, fairness and security possible.

---

## 3. Problem

*What engineering problem does it solve?*

If every program could access memory and devices freely, one bug would corrupt another program's data and one malicious program would own the machine. The kernel exists to make "one program cannot harm another" enforceable by hardware rather than by good manners.

---

## 4. Architecture Position

```text
┌──── USER SPACE ────────────────────────────────┐
│  your app · database · browser · shell          │
└───────────────────┬─────────────────────────────┘
                    │  system calls  (the only doorway)
┌───────────────────▼──── KERNEL SPACE ───────────┐
│  Process management   Memory management         │
│  Scheduler            File systems              │
│  Network stack        Device drivers            │
└───────────────────┬─────────────────────────────┘
                    ↓
              HARDWARE  (CPU, RAM, disk, NIC)
```

> [!IMPORTANT]
> The boundary is enforced by the **CPU itself**, through privilege levels (ring 0 for the kernel, ring 3 for user code). It is not a software convention that a clever program can talk its way past.

---

## 5. Real World Example

- **Linux kernel** — runs virtually every server, every container and every Android phone.
- **[Docker](../13%20-%20DevOps%20and%20Delivery/Docker.md)** — containers are ordinary processes isolated by kernel features (namespaces for visibility, cgroups for resource limits). All containers on a host **share one kernel**, which is why they are fast and why kernel vulnerabilities matter so much to them.
- **The OOM killer** — when memory runs out, the Linux kernel picks a process and kills it. Anyone who has seen a container die with exit code 137 has met this code path.

---

## 6. Input

*What does it receive as input?*

- **[System calls](System%20Calls.md)** from user programs — "open this file", "send these bytes", "give me memory"
- **Hardware interrupts** — "a packet arrived", "the disk finished", "the timer fired"
- **Exceptions** — page faults, division by zero, illegal instruction

---

## 7. Processing

*What happens inside it?*

It validates the request, checks permissions, performs the privileged operation, and returns. Its major subsystems:

| Subsystem | Job |
|---|---|
| **[Scheduler](Scheduler.md)** | Which process runs on which core, and for how long |
| **[Memory manager](Memory%20Management.md)** | Virtual→physical address mapping, paging, swapping |
| **[File systems](File%20Systems.md)** | Files and directories on top of raw blocks |
| **Network stack** | TCP/IP, routing, sockets |
| **Device drivers** | Speaking each device's specific protocol |
| **Security** | Users, permissions, capabilities, namespaces |

---

## 8. Output

*What does it return?*

A result and a status code back to the calling program, plus side effects on hardware: bytes written, packets sent, memory mapped, a process started or killed.

---

## 9. Internal Idea

*How does it work internally?*

The kernel is mostly **event-driven**. It is not constantly running — it sits idle until something needs it:

```text
Something happens
    ↓
System call, interrupt or exception
    ↓
CPU switches to kernel mode
    ↓
Kernel handles it
    ↓
CPU switches back to user mode
```

Two designs exist. A **monolithic kernel** (Linux) keeps drivers and file systems inside the kernel — fast, but a driver bug can crash everything. A **microkernel** keeps only the minimum inside and pushes the rest to user space — safer, but slower because of the extra boundary crossings.

---

## 10. Communication

- **[User space](User%20Space.md)** — via system calls, signals and `/proc`
- **Hardware** — via drivers and interrupts
- **Other kernels** — over the network, as ordinary machines

---

## 11. Dependencies

- CPU support for **privilege levels** and a **memory management unit** (MMU)
- A bootloader (GRUB, systemd-boot) to load it at startup
- Drivers for the hardware present

---

## 12. Alternatives

```text
Monolithic kernel   Linux, Windows NT     fast, large trusted codebase
    ↓
Hybrid              macOS (XNU)           a compromise
    ↓
Microkernel         QNX, seL4, Minix      minimal core, safer, slower
    ↓
Unikernel           one app + kernel compiled together, no isolation needed
```

---

## 13. When To Use

> [!TIP]
> You never "use" the kernel directly, but you should think about it whenever behaviour cannot be explained by your code: processes killed unexpectedly, connection limits, permission errors, or a container behaving differently from your laptop.

---

## 14. When NOT To Use

> [!CAUTION]
> Writing kernel modules or drivers is essentially never the right answer for application problems. Code in the kernel has no safety net — a bug does not throw an exception, it takes down the machine. Solve it in user space unless you are genuinely writing an operating system.

---

## 15. Advantages

- Enforced isolation between processes, backed by hardware
- One implementation of hardware access that all programs share
- Fair resource sharing and enforceable limits
- Centralised security policy

---

## 16. Disadvantages

- Crossing into the kernel costs time — this is why heavy I/O is expensive
- A kernel bug is catastrophic rather than contained
- Monolithic kernels have an enormous trusted codebase
- Containers share the kernel, so a kernel exploit escapes container isolation

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Each system call costs roughly 100 ns – 1 µs of overhead |
| **Memory** | The kernel reserves its own memory and page tables |
| **CPU** | Context switching and interrupt handling are pure overhead |
| **GPU** | Access is mediated by kernel drivers |
| **Network** | The entire TCP/IP stack lives here; high-throughput systems sometimes bypass it (DPDK, io_uring) |

---

## 18. Security Considerations

> [!CAUTION]
> The kernel is the **highest-value target on any machine**. A kernel vulnerability means total compromise — it defeats user permissions, container isolation and process separation simultaneously.

- **Privilege escalation** bugs turn a limited user into root
- **Container escapes** almost always exploit the shared kernel
- **Keep it patched** — kernel updates are among the few that genuinely justify downtime
- Hardening tools: SELinux, AppArmor, seccomp (which restricts which system calls a process may make)

---

## 19. Mental Model

> [!NOTE]
> **The kernel is the building manager who holds the only keys.**
>
> Tenants cannot touch the electrical panel or enter each other's flats. Everything goes through the manager, who checks whether you are allowed before doing it for you. Slower than doing it yourself — and the reason the building is still standing.

---

## 20. Mini Architecture Diagram

```text
Application
    ↓  open(), read(), send()
System call interface
    ↓
┌─────────── KERNEL ───────────┐
│ Scheduler   Memory manager   │
│ VFS         Network stack    │
│ Drivers                      │
└──────────────┬───────────────┘
               ↓
           Hardware
```

---

## 21. Complete Request Flow

Reading a file from an application:

```text
App calls read(fd, buffer, size)
    ↓
CPU switches to kernel mode  (privilege level change)
    ↓
Kernel checks the file descriptor and permissions
    ↓
Virtual file system routes to the right file system driver
    ↓
Is it in the page cache?  → YES → copy to the app's buffer
    ↓ NO
Block layer → disk driver → physical read
    ↓
Data cached, copied into user space
    ↓
CPU switches back to user mode, read() returns
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> The kernel is the only code with real power over the machine; everything your program does to the outside world is a request it makes on your behalf.

---

## 23. Common Mistakes

- **Assuming containers are fully isolated** — they share the host kernel
- **Ignoring kernel-level limits** (file descriptors, memory cgroups, connection backlogs)
- **Blaming the application** for OOM kills that are kernel decisions
- **Skipping kernel security updates** because a reboot is inconvenient
- **Making excessive small system calls** instead of buffering, then wondering why I/O is slow

---

## 24. Open Source Technologies

- **Linux** — the kernel that runs the internet
- **FreeBSD**, **OpenBSD** — alternative Unix kernels with different priorities
- **seL4** — a formally verified microkernel
- **eBPF** — run sandboxed programs inside the Linux kernel safely; the basis of modern observability tools
- **strace**, **bpftrace**, **perf** — watch kernel interactions in real time

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Run `strace` (Linux) on a simple program and count how many system calls it makes.
- [ ] Find the open file descriptor limit on your system and work out what would happen at 10,000 concurrent connections.
- [ ] Explain in two sentences why a kernel vulnerability breaks container isolation.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
User space applications
    ↓
System calls
    ↓
KERNEL
    ↓
Hardware
```

## 2. Request Flow

```text
Input       a system call, interrupt or exception
    ↓
Processing  validate → check permissions → perform the privileged operation
    ↓
Output      a result to the caller, plus a hardware side effect
```

## 3. Real-World Usage

**Docker containers** are ordinary Linux processes wrapped in kernel namespaces and cgroups. The speed advantage over virtual machines comes entirely from sharing one kernel instead of booting another — and so does the security trade-off.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The privileged core of the operating system |
| **Why does it exist?** | So that hardware access and isolation are enforced, not trusted |
| **Where does it belong?** | Between user space and hardware |
| **When should I use it?** | Understand it whenever behaviour cannot be explained by your own code |

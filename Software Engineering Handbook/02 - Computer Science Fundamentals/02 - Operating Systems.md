# Operating Systems

> **In one line —** the program that owns the hardware and rents it out to every other program, safely and fairly.

| | |
|---|---|
| **Category** | Overview note *(hub for this section)* |
| **Architectural Layer** | System Software |
| **Sub-topics** | [Kernel](Kernel.md) · [User Space](User%20Space.md) · [System Calls](System%20Calls.md) · [File Systems](File%20Systems.md) · [Scheduler](Scheduler.md) · [Memory Management](Memory%20Management.md) |

---

## 1. Short Definition

An operating system is the software layer between hardware and applications. It **owns** the [CPU](CPU.md), [RAM](RAM.md) and disks, and hands them out to programs in controlled portions, so that many programs can share one machine without destroying each other.

---

## 2. Purpose

To provide three things every program needs and no program should implement itself:

- **Abstraction** — files instead of disk blocks, sockets instead of network cards
- **Sharing** — many programs on one CPU, each believing it has the machine to itself
- **Protection** — one program cannot read another's memory or crash the system

---

## 3. Problem

Without an operating system, every program would have to speak directly to every model of disk, network card and keyboard, and any bug in any program could take down the whole machine.

```text
Without an OS                      With an OS

App → specific disk model          App → open("file.txt")
App → specific network card                ↓
one bug = machine dead             OS translates and protects
```

---

## 4. The core responsibilities

| Responsibility | What it does | Note |
|---|---|---|
| **Process management** | Start, stop, isolate programs | [Process](Process.md) |
| **Scheduling** | Decide who gets the CPU next | [Scheduler](Scheduler.md) |
| **Memory management** | Give each process its own address space | [Memory Management](Memory%20Management.md) |
| **File systems** | Turn blocks into files and directories | [File Systems](File%20Systems.md) |
| **Device drivers** | Speak to specific hardware | |
| **Networking** | Implement TCP/IP, sockets | [TCP IP](../04%20-%20Networking%20and%20Internet/TCP%20IP.md) |
| **Security** | Users, permissions, isolation | [Authorization](../09%20-%20Security/Authorization.md) |

---

## 5. Architecture Position

The OS is the layer everything else stands on.

```text
Your application  (FastAPI, React build, PostgreSQL)
    ↓
Runtime / Interpreter  (CPython, Node.js, JVM)
    ↓
Standard library
    ↓
System calls                ← the doorway between the two worlds
    ↓
KERNEL   (scheduler, memory manager, file systems, drivers)
    ↓
Hardware  (CPU, RAM, disk, network card)
```

---

## 6. The two worlds

> [!IMPORTANT]
> The single most important idea in operating systems is the split between **[user space](User%20Space.md)** and **[kernel space](Kernel.md)**. Your code runs in user space with no direct access to hardware; every time it needs something real — a file, a socket, more memory — it must ask the kernel through a [system call](System%20Calls.md).

```text
┌──── USER SPACE ──── restricted, isolated, crashes affect one process ────┐
│  your app · browser · database · shell                                    │
└───────────────────────────┬───────────────────────────────────────────────┘
                            │  system calls
┌───────────────────────────▼─── KERNEL SPACE ── full hardware access ─────┐
│  scheduler · memory manager · file systems · drivers · network stack      │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Real World Example

- **Linux** runs essentially all servers, all containers and all of Android. If you deploy anything, you deploy onto Linux.
- **[Docker](../13%20-%20DevOps%20and%20Delivery/Docker.md)** is not a virtual machine — it uses Linux kernel features (namespaces and cgroups) to isolate processes. Containers share the host kernel, which is why they start in milliseconds.
- **Kubernetes** is, at bottom, a system for deciding which kernel on which machine runs which process.

---

## 8. Why this matters to a backend developer

Most production mysteries are operating-system behaviour:

- "Why did my container get killed?" → the kernel's out-of-memory killer
- "Why is the app slow but the CPU idle?" → blocked on I/O
- "Why can only 1024 connections open?" → file descriptor limits
- "Why does it work locally but not in Docker?" → different kernel, different limits, different file system

---

## 9. Mental Model

> [!NOTE]
> **The OS is the building manager of an apartment block.**
>
> Tenants (processes) do not own the plumbing or the electricity. They ask the manager for what they need, they cannot enter each other's flats, and if one floods their bathroom the manager contains the damage rather than letting the building collapse.

---

## 10. Key Takeaway

> [!IMPORTANT]
> The operating system owns the hardware and lends it out under supervision; every interaction between your code and the real machine passes through it.

---

## 11. Common Mistakes

- Believing your program talks to hardware directly
- Ignoring OS-level limits — memory, file descriptors, process counts — until production hits them
- Assuming containers are virtual machines
- Treating "the server is slow" as an application problem before checking what the OS says

---

## 12. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 13. Workbook Exercise

- [ ] Run `top` (or Task Manager) and identify which of your processes is using CPU, and which is waiting.
- [ ] Find the memory limit of a container you run, and what happens when it is exceeded.
- [ ] Draw the user space / kernel space boundary and mark where a database query crosses it.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Application
    ↓
Runtime
    ↓
System calls
    ↓
Kernel
    ↓
Hardware
```

## 2. Request Flow

```text
Input       a program's request for a resource
    ↓
Processing  the kernel validates, schedules and performs it
    ↓
Output      the result, plus enforced isolation from every other process
```

## 3. Real-World Usage

**Docker** exists because of Linux kernel namespaces and cgroups. Container isolation is not a Docker invention — it is an operating-system feature that Docker made usable.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The software that owns the hardware and shares it among programs |
| **Why does it exist?** | To provide abstraction, sharing and protection |
| **Where does it belong?** | Between hardware and every runtime |
| **When should I use it?** | Always — the question is only how much of it you understand |

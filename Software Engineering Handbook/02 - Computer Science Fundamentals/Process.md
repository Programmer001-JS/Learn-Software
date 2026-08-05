# Process

> **In one line —** a running program together with everything the operating system gives it: its own memory, its own file handles, and its own isolation.

| | |
|---|---|
| **Category** | Operating System Concept |
| **Architectural Layer** | User Space |
| **Related notes** | [Thread](Thread.md) · [Processes and Threads](03%20-%20Processes%20and%20Threads.md) · [Scheduler](Scheduler.md) · [Memory Management](Memory%20Management.md) · [Docker](../13%20-%20DevOps%20and%20Delivery/Docker.md) |

---

## 1. Short Definition

*What is it?*

A process is an instance of a program in execution. The program is the file on disk; the process is what exists once it is loaded into memory and given a private address space, an ID, and a share of the CPU.

---

## 2. Purpose

*What is its main purpose?*

To be the operating system's **unit of isolation**. Everything the OS protects — memory, file descriptors, permissions, resource limits — is tracked per process.

---

## 3. Problem

*What engineering problem does it solve?*

Many programs must share one machine without being able to read, corrupt or crash each other. The process is the box that makes that guarantee enforceable.

```text
Program on disk (passive)
    ↓  execute
Process (active): own memory, own PID, own file descriptors, own permissions
```

---

## 4. Architecture Position

```text
Operating System
    ↓
┌── Process 1 ──┐  ┌── Process 2 ──┐  ┌── Process 3 ──┐
│ virtual memory│  │ virtual memory│  │ virtual memory│
│ file descrs   │  │ file descrs   │  │ file descrs   │
│ threads       │  │ threads       │  │ threads       │
└───────────────┘  └───────────────┘  └───────────────┘
        ↓                  ↓                  ↓
              Scheduler → CPU cores
```

---

## 5. Real World Example

- **Chrome** — one process per tab, so a crashing page cannot take down the browser or read another tab's memory.
- **PostgreSQL** — one backend process per connection; a crash in one is contained and does not corrupt the server.
- **Gunicorn / Uvicorn** — several worker processes, because Python's GIL prevents CPU parallelism inside a single process.
- **[Docker](../13%20-%20DevOps%20and%20Delivery/Docker.md) containers** — a container is a process (or a small group) with extra kernel isolation applied.

---

## 6. Input

*What does it receive as input?*

- An **executable** and its arguments
- **Environment variables**
- **Standard input**, plus any files or sockets it opens
- **Signals** from the OS or other processes (`SIGTERM`, `SIGKILL`)

---

## 7. Processing

*What happens inside it?*

The OS creates a process control block, sets up a private virtual address space, loads the executable, and starts its main [thread](Thread.md). From there the process runs, alternating between computing and waiting on I/O.

```text
NEW  →  READY  →  RUNNING  →  TERMINATED
                     ↕
                  BLOCKED  (waiting on I/O)
```

---

## 8. Output

*What does it return?*

An **exit code** — 0 for success, non-zero for failure — plus whatever it wrote to files, sockets, or standard output. Exit codes matter operationally: `137` means killed by SIGKILL (usually the OOM killer), `143` means terminated by SIGTERM.

---

## 9. Internal Idea

*How does it work internally?*

The kernel keeps a structure per process holding its ID, parent, memory map, open file descriptors, permissions and scheduling state. A [context switch](Context%20Switching.md) is essentially saving and restoring that state.

New processes are created by `fork()` (duplicate the current one) followed by `execve()` (replace its program). The duplicate is cheap because pages are shared **copy-on-write** until one side writes.

---

## 10. Communication

Processes cannot read each other's memory, so they talk through kernel-mediated channels:

- **Pipes** — `cmd1 | cmd2`
- **Sockets** — including over the network; how microservices talk
- **Shared memory** — fastest, but requires explicit synchronisation
- **Signals** — simple notifications
- **Files** and **message queues**

---

## 11. Dependencies

- An [operating system](02%20-%20Operating%20Systems.md) to create and schedule it
- [Memory](Memory%20Management.md) and a CPU share
- An executable and its runtime or shared libraries

---

## 12. Alternatives

```text
Process       full isolation, expensive to create
    ↓
Thread        shared memory, cheap, no isolation
    ↓
Coroutine     goroutines, async/await — user-space, extremely cheap
    ↓
Container     a process plus kernel namespaces and limits
    ↓
VM            a whole guest kernel; strongest isolation, heaviest
```

---

## 13. When To Use

> [!TIP]
> Use separate processes when you need **isolation** or genuine **CPU parallelism** — especially in Python, where the GIL means processes are the only way to use multiple cores for computation.

---

## 14. When NOT To Use

> [!CAUTION]
> Do not spawn a process per request or per task. Creation costs milliseconds and megabytes; at any real request rate this collapses. Use a **fixed pool** of worker processes instead.

---

## 15. Advantages

- Complete memory isolation, enforced by hardware
- A crash affects only itself
- True parallelism across cores, with no GIL restriction
- Independent resource limits and permissions per process

---

## 16. Disadvantages

- Expensive to create — milliseconds and a full address space
- Higher memory usage; each has its own copy of runtime and libraries
- Inter-process communication is slow compared with shared memory
- Coordinating many processes adds real operational complexity

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Creation ~1–10 ms; IPC far slower than a shared variable |
| **Memory** | Each carries its own runtime — a Python worker easily costs 50–200 MB |
| **CPU** | Real parallelism across cores |
| **GPU** | Each process needs its own GPU context; VRAM is quickly exhausted by several |
| **Network** | Processes on different machines communicate exactly like local ones — over sockets |

---

## 18. Security Considerations

> [!CAUTION]
> Process isolation is the foundation everything else stands on. If it fails — through a kernel bug or a container escape — no application-level security control matters.

- Run as a **non-root user**; a compromise in a root process is a compromise of the machine
- Apply **resource limits** so one process cannot exhaust memory or file descriptors
- Environment variables holding secrets are visible to anyone who can read `/proc/<pid>/environ`
- Containers add namespaces and cgroups, but still share the host [kernel](Kernel.md)

---

## 19. Mental Model

> [!NOTE]
> **A process is a separate company with its own office.**
>
> No other company can walk in and read the filing cabinet. Communication requires a phone call or a courier — slower than shouting across the room, and precisely why nothing that happens in one office can destroy another.

---

## 20. Mini Architecture Diagram

```text
Executable on disk
    ↓  fork + exec
┌──────── PROCESS ────────┐
│  PID, parent PID        │
│  virtual address space  │
│  file descriptors       │
│  permissions, limits    │
│  one or more threads    │
└──────────┬──────────────┘
           ↓
      Scheduler → CPU
```

---

## 21. Complete Request Flow

A web server handling a request with a pre-forked worker model:

```text
Master process starts and binds the port
    ↓
fork() × 4 → four worker processes  (copy-on-write, cheap)
    ↓
Request arrives → the kernel hands the connection to one worker
    ↓
Worker handles it in its own isolated memory
    ↓
Worker crashes?  → only that request fails; the master forks a replacement
    ↓
Response returned
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> A process is the operating system's unit of isolation — expensive to create, impossible for others to reach into, and the only way to get true CPU parallelism in languages with a GIL.

---

## 23. Common Mistakes

- **Spawning a process per request or per job** instead of using a pool
- **Assuming processes share memory** — they do not, and global state does not propagate between workers
- **Ignoring exit codes**, especially 137 (OOM kill) and 143 (SIGTERM)
- **Not handling SIGTERM** — the process gets killed mid-work instead of shutting down cleanly
- **Running as root** in containers without needing to
- **Creating zombie processes** by never reaping children

---

## 24. Open Source Technologies

- **systemd**, **supervisord** — process management and restarts
- **Gunicorn**, **uWSGI**, **PM2** — worker process managers for web applications
- **Docker**, **containerd** — processes with kernel isolation
- **htop**, **ps**, **lsof** — inspect running processes and what they hold open

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Find how many worker processes your application runs and how much memory each uses.
- [ ] Check whether your application handles SIGTERM and shuts down cleanly.
- [ ] Explain in two sentences why PostgreSQL uses one process per connection instead of one thread.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Operating System
    ↓
Process  (isolated memory, own file descriptors)
    ↓
Threads
    ↓
CPU
```

## 2. Request Flow

```text
Input       an executable, arguments and environment
    ↓
Processing  loaded into a private address space, scheduled, run
    ↓
Output      an exit code and whatever it wrote to files and sockets
```

## 3. Real-World Usage

**Chrome** puts every tab in its own process. The memory cost is substantial and entirely deliberate: it is what stops one malicious or broken page from crashing the browser or reading another tab's data.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A running program with its own isolated memory and resources |
| **Why does it exist?** | To let many programs share a machine without harming each other |
| **Where does it belong?** | Between the operating system and the threads that do the work |
| **When should I use it?** | For isolation, fault containment, and real CPU parallelism |

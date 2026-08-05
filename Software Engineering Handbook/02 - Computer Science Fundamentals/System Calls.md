# System Calls

> **In one line —** the only doorway between your program and the real machine; every file, socket and byte of memory comes through one.

| | |
|---|---|
| **Category** | System Software Concept |
| **Architectural Layer** | User Space ↔ Kernel Space boundary |
| **Related notes** | [Kernel](Kernel.md) · [User Space](User%20Space.md) · [Process](Process.md) · [File Systems](File%20Systems.md) · [Context Switching](Context%20Switching.md) |

---

## 1. Short Definition

*What is it?*

A system call is a request from a user-space program asking the [kernel](Kernel.md) to perform a privileged operation on its behalf — opening a file, sending a packet, allocating memory, creating a process.

---

## 2. Purpose

*What is its main purpose?*

To provide a **controlled, checkable entry point** into privileged code. Your program cannot touch hardware, so it must ask; the kernel validates every request before performing it.

---

## 3. Problem

*What engineering problem does it solve?*

We need programs to be able to do real work (write files, open sockets) without giving them the power to damage the system. A system call is the compromise: full capability, delivered through a narrow gate with a permission check on it.

---

## 4. Architecture Position

```text
Your code
    ↓
Standard library     fopen(), requests.get(), fs.readFile()
    ↓
SYSTEM CALL          open(), read(), write(), socket()      ← the boundary
    ↓
Kernel
    ↓
Hardware
```

> [!IMPORTANT]
> You almost never call these directly. `open()` in Python, `fs.readFile` in Node and `File.read` in Java all eventually become the same handful of kernel calls.

---

## 5. Real World Example

- **Every HTTP request** your server handles involves `accept`, `read`, `write` and `close` at minimum.
- **`strace`** on a running process shows exactly this stream and is one of the most effective debugging tools that exists — it reveals what a program *actually* does, regardless of what the code appears to say.
- **Docker security** uses **seccomp** profiles to forbid dangerous system calls inside containers.

---

## 6. Input

*What does it receive as input?*

A system call **number** identifying the operation, plus arguments in [registers](Registers.md) — a file descriptor, a buffer address, a length, flags.

---

## 7. Processing

*What happens inside it?*

```text
Program executes a special instruction  (syscall / int 0x80)
    ↓
CPU switches from user mode to kernel mode
    ↓
Kernel looks up the handler in the syscall table
    ↓
Validates arguments and permissions     ← never trusts user-supplied pointers
    ↓
Performs the operation
    ↓
Copies the result back to user space
    ↓
CPU switches back to user mode
```

---

## 8. Output

*What does it return?*

An integer result: a file descriptor, a byte count, or a negative error code (`EACCES`, `ENOENT`, `EAGAIN`). Standard libraries translate these into exceptions or error objects in your language.

---

## 9. Internal Idea

*How does it work internally?*

The CPU provides a dedicated instruction that changes privilege level and jumps to a fixed kernel entry point. The program cannot choose where it lands — the kernel does. That is what makes the gate safe.

---

## 10. The main categories

| Category | Examples |
|---|---|
| **Files** | `open`, `read`, `write`, `close`, `stat` |
| **Processes** | `fork`, `execve`, `wait`, `exit`, `kill` |
| **Memory** | `mmap`, `munmap`, `brk` |
| **Network** | `socket`, `bind`, `listen`, `accept`, `send`, `recv` |
| **Time** | `clock_gettime`, `nanosleep` |
| **Sync** | `futex`, `poll`, `epoll`, `io_uring` |

---

## 11. Dependencies

- CPU support for privilege levels and a syscall instruction
- A kernel with a syscall table
- Usually a standard library (libc) providing a usable wrapper

---

## 12. Alternatives

```text
Traditional syscalls    one crossing per operation
    ↓
Batched interfaces      epoll, io_uring — many operations per crossing
    ↓
vDSO                    a few calls (like clock_gettime) served without entering the kernel
    ↓
Kernel bypass           DPDK, SPDK — talk to hardware from user space, for extreme throughput
```

---

## 13. When To Use

> [!TIP]
> You do not choose to make system calls — you choose **how many**. The practical rule: buffer your I/O. One 64 KB write beats 64,000 one-byte writes by orders of magnitude.

---

## 14. When NOT To Use

> [!CAUTION]
> Avoid making system calls inside tight loops. Reading a file byte by byte, calling `time()` per iteration, or writing unbuffered log lines per request are all common and expensive mistakes.

---

## 15. Advantages

- Safe, validated access to privileged operations
- A stable interface — Linux takes backwards compatibility here very seriously
- Language-independent: every runtime uses the same set
- Observable and restrictable (`strace`, `seccomp`)

---

## 16. Disadvantages

- Each crossing costs roughly 100 ns – 1 µs
- Data must be copied between user and kernel buffers
- Blocking calls stall the calling thread
- Spectre/Meltdown mitigations made crossings measurably more expensive

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Individually small, but they dominate I/O-heavy workloads |
| **Memory** | Copying between kernel and user buffers |
| **CPU** | Mode switches flush pipelines and pollute caches |
| **GPU** | Driver access is syscall-mediated |
| **Network** | The main reason `epoll` and `io_uring` exist — fewer crossings per connection |

---

## 18. Security Considerations

> [!CAUTION]
> The syscall interface is the **attack surface of the kernel**. Most privilege-escalation exploits work by passing carefully malformed arguments to an obscure system call.

- **seccomp** restricts which calls a process may make — Docker applies a default profile, and tightening it is one of the highest-value container hardening steps
- The kernel must validate every user-supplied pointer; historically, failures to do so were exactly the bug class attackers used
- `strace` on a suspicious process is a fast way to see what it is really doing

---

## 19. Mental Model

> [!NOTE]
> **A system call is a form submitted at a government counter.**
>
> You cannot walk into the archive yourself. You fill in the request, hand it over, the clerk checks whether you are entitled to it, fetches it and hands it back. Slower than doing it yourself — and the reason the archive still exists.

---

## 20. Mini Architecture Diagram

```text
Application code
    ↓
Language runtime / libc
    ↓
┌──────── syscall instruction ────────┐   ← privilege switch
    ↓
Kernel syscall handler
    ↓
Subsystem (VFS, network stack, memory manager)
    ↓
Hardware
```

---

## 21. Complete Request Flow

Reading a configuration file:

```text
open("config.json")
    ↓  syscall #2  → kernel checks permissions → returns fd = 3
read(3, buffer, 4096)
    ↓  syscall #0  → page cache hit → 4096 bytes copied to user space
close(3)
    ↓  syscall #3  → kernel releases the descriptor
```

Three crossings for one small file. Now imagine doing that per log line, per request, at 5,000 requests per second — this is why buffering matters.

---

## 22. Key Takeaway

> [!IMPORTANT]
> Every real thing your program does is a system call, and each one costs a privilege switch — so make fewer, larger ones.

---

## 23. Common Mistakes

- **Unbuffered I/O** — one syscall per byte or per line
- **Blocking calls on the main thread** in a server, stalling everything
- **Ignoring error codes**, especially `EAGAIN` and `EINTR`
- **Leaking file descriptors** — every unclosed socket or file is a leaked kernel resource, and the limit is real
- **Not knowing `strace` exists** — it answers "what is this process actually doing?" in seconds

---

## 24. Open Source Technologies

- **strace**, **ltrace**, **dtrace**, **bpftrace** — observe system calls live
- **seccomp**, **gVisor** — restrict or intercept them for sandboxing
- **io_uring** — modern asynchronous I/O with far fewer crossings
- **musl**, **glibc** — the wrappers your language actually calls

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Run `strace -c` on a simple script and look at which system calls dominate.
- [ ] Find one place in your code doing unbuffered I/O and fix it.
- [ ] Explain in two sentences why `epoll` exists.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Application
    ↓
Standard library
    ↓
SYSTEM CALL
    ↓
Kernel
    ↓
Hardware
```

## 2. Request Flow

```text
Input       a syscall number and arguments in registers
    ↓
Processing  privilege switch → validate → perform → copy result back
    ↓
Output      a return value or an error code
```

## 3. Real-World Usage

**Nginx** handles tens of thousands of concurrent connections on one machine by using `epoll` to learn about many sockets in a single system call, instead of one blocking call per connection. The architecture is a direct response to the cost of crossing this boundary.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A program's request for the kernel to perform a privileged operation |
| **Why does it exist?** | To allow real work without granting real power |
| **Where does it belong?** | Exactly on the user space / kernel space boundary |
| **When should I use it?** | Constantly and indirectly — the skill is minimising how many you trigger |

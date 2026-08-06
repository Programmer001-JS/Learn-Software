# Virtual Machines

> **In one line —** a machine implemented in software; the word means two genuinely different things, and confusing them causes real misunderstandings.

| | |
|---|---|
| **Category** | Runtime / Infrastructure Concept |
| **Architectural Layer** | Language runtime, or infrastructure |
| **Related notes** | [Bytecode](Bytecode.md) · [JVM](JVM.md) · [.NET Runtime](dotNET%20Runtime.md) · [Docker](../13%20-%20DevOps%20and%20Delivery/Docker.md) · [EC2](../12%20-%20Cloud%20Architecture/EC2.md) · [Kernel](../02%20-%20Computer%20Science%20Fundamentals/Kernel.md) |

---

## 1. Two different things with the same name

> [!IMPORTANT]
> **Process VM** — runs *one program* written in bytecode. Examples: the [JVM](JVM.md), the [.NET runtime](dotNET%20Runtime.md), [CPython](CPython.md).
>
> **System VM** — runs *a whole operating system* on virtualised hardware. Examples: VMware, VirtualBox, AWS [EC2](../12%20-%20Cloud%20Architecture/EC2.md), Hyper-V.
>
> They share a name and almost nothing else. This note covers both, kept clearly apart.

```text
PROCESS VM                        SYSTEM VM

Your bytecode                     Guest OS + its applications
    ↓                                 ↓
JVM / CLR / CPython               Virtual hardware (CPU, RAM, disk, NIC)
    ↓                                 ↓
Host operating system             Hypervisor
    ↓                                 ↓
Hardware                          Physical hardware
```

---

# Part 1 — Process Virtual Machines

## 2. Short Definition

A process VM is a program that **executes [bytecode](Bytecode.md)** as if it were a CPU. It provides an instruction set, memory management, and usually garbage collection, so the same bytecode behaves identically on every platform.

---

## 3. Purpose

To make one compiled artifact run anywhere, and to allow runtime optimisation (JIT) that an ahead-of-time compiler cannot perform.

---

## 4. What it provides

- **Execution** — interpret bytecode, and JIT-compile the parts that run often
- **Memory management** — allocation plus garbage collection
- **Safety** — bounds checking, type verification, no raw pointers
- **Platform abstraction** — one API over many operating systems

---

## 5. Real World Example

- **JVM** — runs Java, Kotlin, Scala and Clojure; the same `.jar` works on a server and a phone.
- **.NET CLR** — runs C# and F# across Windows, Linux and macOS.
- **V8** — executes JavaScript in Chrome and in [Node.js](Node.js%20Runtime.md).
- **CPython** — the reference Python implementation.

---

# Part 2 — System Virtual Machines

## 6. Short Definition

A system VM emulates an entire computer — CPU, memory, disks, network cards — so that a complete guest operating system can run inside it, believing it owns real hardware.

---

## 7. Purpose

To run several isolated operating systems on one physical machine. This is the foundation the entire cloud industry was built on: [EC2](../12%20-%20Cloud%20Architecture/EC2.md) instances are virtual machines.

---

## 8. Architecture Position

```text
┌──── VM 1 ────┐  ┌──── VM 2 ────┐  ┌──── VM 3 ────┐
│  Guest OS     │  │  Guest OS     │  │  Guest OS     │
│  apps         │  │  apps         │  │  apps         │
└───────┬───────┘  └───────┬───────┘  └───────┬───────┘
        └──────────────────┼──────────────────┘
                    HYPERVISOR
                           ↓
                  Physical hardware
```

A **type 1** hypervisor (KVM, Xen, ESXi) runs directly on hardware and is what clouds use. A **type 2** hypervisor (VirtualBox, VMware Workstation) runs as an application on a normal desktop OS.

---

## 9. Virtual Machines vs Containers

This comparison matters more in practice than any other point in this note.

```text
VIRTUAL MACHINE                   CONTAINER

App                               App
Libraries                         Libraries
GUEST KERNEL          ← the difference →   (none — shares the host kernel)
Virtual hardware
Hypervisor                        Container runtime
Host OS                           Host OS
Hardware                          Hardware
```

| | Virtual Machine | [Container](../13%20-%20DevOps%20and%20Delivery/Docker.md) |
|---|---|---|
| **Boot time** | 30–60 seconds | Milliseconds |
| **Size** | Gigabytes | Megabytes |
| **Isolation** | Very strong — separate kernel | Weaker — shared kernel |
| **Overhead** | 5–15% | Near zero |
| **Different OS** | Yes — Windows on Linux | No — same kernel only |
| **Density per host** | Tens | Hundreds |

> [!IMPORTANT]
> The single sentence to remember: **a VM virtualises the hardware, a container virtualises the operating system.** Everything else in the table follows from that.

---

## 10. When To Use

> [!TIP]
> **System VM** — when you need strong isolation between untrusted workloads, a different operating system, or a full machine you control (kernel modules, custom drivers).
>
> **Container** — for everything else in modern application deployment: faster, smaller, denser.

---

## 11. When NOT To Use

> [!CAUTION]
> Do not run a full VM per microservice. The gigabytes of memory and the minute of boot time buy you isolation you probably do not need. Conversely, do not rely on container isolation for genuinely hostile multi-tenant workloads — a shared [kernel](../02%20-%20Computer%20Science%20Fundamentals/Kernel.md) is a shared attack surface.

---

## 12. Advantages and Disadvantages

**System VMs**
- Strongest practical isolation short of separate hardware
- Can run any operating system
- Snapshots and live migration
- ✗ Heavy: gigabytes of RAM, slow boot, 5–15% overhead

**Process VMs**
- Portability and runtime optimisation
- Memory safety and garbage collection
- ✗ Startup cost, memory footprint, must be shipped with the app

---

## 13. Performance Impact

| Resource | System VM | Process VM |
|---|---|---|
| **Speed** | 5–15% overhead | Near-native after JIT warm-up |
| **Memory** | Full guest OS per VM | Runtime baseline plus heap |
| **CPU** | Hardware-assisted, so modest cost | JIT compilation during warm-up |
| **Startup** | 30–60 s | 100 ms – several seconds |
| **Density** | Tens per host | Irrelevant |

---

## 14. Security Considerations

> [!CAUTION]
> VM isolation is strong but not absolute. **VM escape** vulnerabilities exist, and side-channel attacks (Spectre, L1TF) have read across VM boundaries on shared cloud hardware without any software bug in the hypervisor.

- Keep hypervisors patched — a compromise exposes every guest on the host
- For genuinely hostile multi-tenancy, VMs remain the stronger boundary than containers
- Snapshots contain memory contents, including secrets in plaintext
- **gVisor** and **Firecracker** exist precisely to combine container speed with VM-grade isolation

---

## 15. Mental Model

> [!NOTE]
> **A system VM is a self-contained apartment inside a house** — its own kitchen, plumbing and front door. Expensive, but genuinely separate.
>
> **A container is a locked room in a shared flat** — its own space, but the kitchen and plumbing (the kernel) belong to everyone.
>
> **A process VM is a translator who reads a universal recipe** and cooks it in whatever kitchen is available.

---

## 16. Complete Request Flow

A request reaching an application in the cloud:

```text
Request arrives at the physical machine
    ↓
Hypervisor routes it to the correct VM's virtual network card
    ↓
Guest OS kernel receives it
    ↓
Container runtime (if any) routes it to the container
    ↓
Process VM (JVM / CPython) executes the application bytecode
    ↓
Response travels back down the same stack
```

> [!IMPORTANT]
> A typical cloud request often crosses **all three** kinds of virtualisation. Knowing which layer is which is what lets you diagnose where the latency actually comes from.

---

## 17. Key Takeaway

> [!IMPORTANT]
> A system VM virtualises hardware and runs a whole OS; a process VM virtualises a CPU and runs bytecode. Containers are neither — they share the host kernel, which is why they are fast and less isolated.

---

## 18. Common Mistakes

- **Calling a container a VM** — the difference is the guest kernel, and it changes everything
- **Assuming container isolation equals VM isolation**
- **Running a VM per microservice** and paying enormous overhead
- **Ignoring hypervisor patching** in self-hosted environments
- **Forgetting a process VM ships with the application**, inflating image sizes

---

## 19. Open Source Technologies

- **KVM**, **Xen**, **QEMU** — hypervisors behind most cloud infrastructure
- **Firecracker** — micro-VMs powering AWS Lambda; VM isolation with millisecond boot
- **gVisor** — a user-space kernel that hardens container isolation
- **OpenJDK**, **CPython**, **.NET**, **V8** — the process VMs you will actually use

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Draw the layer stack for a VM and for a container side by side, and mark where the kernel sits in each.
- [ ] Work out whether your current deployment runs on VMs, containers, or containers inside VMs.
- [ ] Explain in two sentences why AWS Lambda uses Firecracker micro-VMs rather than plain containers.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Application bytecode
    ↓
Process VM (JVM / CLR / CPython)
    ↓
Container runtime (optional)
    ↓
Guest OS
    ↓
Hypervisor
    ↓
Physical hardware
```

## 2. Request Flow

```text
Input       bytecode, or a whole operating system image
    ↓
Processing  emulated CPU and memory, or virtualised hardware
    ↓
Output      execution isolated from everything else on the machine
```

## 3. Real-World Usage

**AWS Lambda** runs each function inside a **Firecracker** micro-VM. AWS needed container-like start times with VM-grade isolation between customers, and building a minimal hypervisor was the answer.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A machine implemented in software — either a CPU for bytecode, or a whole computer |
| **Why does it exist?** | For portability (process VM) and isolation plus consolidation (system VM) |
| **Where does it belong?** | Between application and OS, or between OS and hardware |
| **When should I use it?** | Process VMs come with your language; system VMs when isolation must be strong |

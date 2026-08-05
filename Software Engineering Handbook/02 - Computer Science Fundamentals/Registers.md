# Registers

> **In one line —** the handful of tiny storage slots physically inside the CPU where every instruction's operands and results actually live.

| | |
|---|---|
| **Category** | Hardware Component |
| **Architectural Layer** | Physical / Compute |
| **Size** | ~16–32 slots of 64 bits each |
| **Related notes** | [CPU](CPU.md) · [Cache Memory](Cache%20Memory.md) · [RAM](RAM.md) · [Context Switching](Context%20Switching.md) · [Binary and Machine Code](Binary%20and%20Machine%20Code.md) |

---

## 1. Short Definition

*What is it?*

Registers are the small named storage locations built directly into the [CPU](CPU.md). A typical 64-bit processor has a few dozen of them, each holding 8 bytes. They are the **only** place the CPU can operate on data directly.

---

## 2. Purpose

*What is its main purpose?*

To hold the values an instruction is working on **right now**. Every arithmetic operation, comparison and address calculation happens between registers — never directly in RAM.

---

## 3. Problem

*What engineering problem does it solve?*

The CPU's logic units need their inputs available in a fraction of a nanosecond. Even the [L1 cache](Cache%20Memory.md) is too far away for that. Registers are physically part of the execution circuitry, which is the only way to feed a processor at full speed.

---

## 4. Architecture Position

Registers are the **top** of the memory hierarchy — the smallest and fastest storage in the machine.

```text
Registers      ~1 KB total     ~0.3 ns    ← you are here
    ↓
L1 Cache       ~64 KB          ~1 ns
    ↓
L2 / L3 Cache  ~MB             ~4–40 ns
    ↓
RAM            ~GB             ~100 ns
    ↓
Disk           ~TB             ~100 µs+
```

---

## 5. Real World Example

You never write to registers directly in application code, but you rely on them constantly:

- **Compilers** spend significant effort on *register allocation* — deciding which variables live in registers rather than memory. This is one of the largest sources of optimisation in compiled code.
- **Debuggers** show register contents when a program crashes; the instruction pointer register tells you exactly where it died.
- **[Context switching](Context%20Switching.md)** is fundamentally the act of saving and restoring all registers — that is what makes it expensive.

---

## 6. Input

*What does it receive as input?*

Values loaded from [cache](Cache%20Memory.md) or [RAM](RAM.md) by explicit load instructions, results produced by the ALU, or constants encoded directly in an instruction.

---

## 7. Processing

*What happens inside it?*

Registers do not process — they hold. A typical instruction sequence looks like this:

```text
LOAD   R1, [address]      copy from memory into register 1
LOAD   R2, [address]      copy from memory into register 2
ADD    R3, R1, R2         add them, result into register 3
STORE  [address], R3      copy the result back to memory
```

Notice that memory is only ever *loaded from* and *stored to*. All actual work happens between registers.

---

## 8. Output

*What does it return?*

The value it holds, delivered instantly to the arithmetic unit, or written back out to cache and memory by a store instruction.

---

## 9. Internal Idea

*How does it work internally?*

Each register is a small bank of flip-flops — circuits that hold a bit as long as power is applied — wired directly into the execution units. There is no addressing, no lookup and no waiting: the value is simply *there*, at the input of the adder.

Because they are so expensive in silicon area, there are very few of them. That scarcity is why compilers work so hard to decide what deserves one.

---

## 10. Communication

- **The ALU** and control unit inside the [CPU](CPU.md)
- **[L1 cache](Cache%20Memory.md)** — the only memory they exchange data with directly
- **The [operating system](02%20-%20Operating%20Systems.md)** — indirectly, when it saves and restores them during a [context switch](Context%20Switching.md)

---

## 11. Dependencies

- The CPU itself; registers cannot be added, removed or configured
- The **instruction set architecture** (x86-64, ARM64, RISC-V) defines how many exist and what each is named

---

## 12. Alternatives

None — this is the floor of the hierarchy. The only trade-off available to a chip designer is *how many* registers to provide:

```text
Few registers   (classic x86)   → more memory traffic, smaller instructions
Many registers  (ARM, RISC-V)   → less memory traffic, larger instruction encoding
```

---

## 13. When To Use

> [!TIP]
> You do not use registers directly unless you write assembly. What matters for you is understanding that **local variables in a tight loop often live in registers**, which is why they are essentially free compared to anything that touches memory.

---

## 14. When NOT To Use

> [!CAUTION]
> Never try to reason about registers in application-level performance work. The compiler allocates them far better than a human can, and no decision you make in Python or JavaScript reaches this level directly.

---

## 15. Advantages

- The fastest storage that exists — sub-nanosecond
- No addressing or lookup overhead at all
- Directly wired into the execution units

---

## 16. Disadvantages

- Extremely few of them — a few hundred bytes in total
- Not addressable from high-level languages
- Contents must be saved and restored on every [context switch](Context%20Switching.md), which is a real cost
- Register pressure (more live variables than registers) forces "spilling" to memory

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Fastest possible; a register access is effectively free |
| **Memory** | Negligible footprint |
| **CPU** | Register pressure directly affects how fast compiled code runs |
| **GPU** | GPUs have far more registers per core, which is part of how they hide memory latency |
| **Network** | Not involved |

---

## 18. Security Considerations

> [!CAUTION]
> Registers hold secrets in plaintext while they are in use — decryption keys, passwords, tokens. During a [context switch](Context%20Switching.md) they are written into kernel memory, and in a crash dump they end up on disk.

Speculative-execution vulnerabilities such as Spectre worked precisely by observing effects of values that transited through registers and cache.

---

## 19. Mental Model

> [!NOTE]
> **Registers = your hands.**
>
> You can only hold two or three things at once. Everything else sits on the desk ([cache](Cache%20Memory.md)) or the shelf ([RAM](RAM.md)), and you must physically pick it up before you can do anything with it.

---

## 20. Mini Architecture Diagram

```text
        ┌─────────── CPU core ───────────┐
        │                                │
        │   Registers  ←──────→  ALU     │
        │      ↑                         │
        │      │  load / store           │
        │      ↓                         │
        │   L1 Cache                     │
        └────────────┬───────────────────┘
                     ↓
                    RAM
```

---

## 21. Complete Request Flow

```text
Instruction:  "add the two numbers at addresses A and B"
    ↓
LOAD  value at A  →  register R1     (from cache, ~1 ns)
    ↓
LOAD  value at B  →  register R2
    ↓
ADD   R1 + R2     →  register R3     (~0.3 ns, inside the ALU)
    ↓
STORE R3          →  memory address C
```

And during a context switch:

```text
Timer interrupt
    ↓
Kernel saves ALL registers of the current process into its control block
    ↓
Kernel loads the next process's registers
    ↓
Execution resumes exactly where that process left off
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> Registers are the only place the CPU can actually compute; everything else in the memory hierarchy exists to feed them.

---

## 23. Common Mistakes

- **Confusing registers with cache** — cache is memory, registers are part of the execution logic
- **Thinking high-level variables map one-to-one to registers** — the compiler decides, and most do not
- **Underestimating context switch cost** — saving and restoring this state is precisely why switching is expensive
- **Trying to hand-optimise register use** in a language that has no concept of them

---

## 24. Open Source Technologies

- **LLVM**, **GCC** — their register allocators are among the most studied pieces of compiler engineering
- **RISC-V** — an open instruction set, useful for actually reading a register specification
- **GDB**, **LLDB** — inspect real register contents while debugging

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Compile a tiny C function and read its assembly output (`gcc -S`). Identify which values were placed in registers.
- [ ] Draw the memory hierarchy and mark where registers sit, with sizes and timings.
- [ ] Explain in two sentences why a context switch is expensive, using registers in the explanation.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
ALU  ←→  Registers
             ↓
         L1 Cache
             ↓
            RAM
```

## 2. Request Flow

```text
Input       a load instruction with a memory address
    ↓
Processing  value placed in a register; ALU operates on registers only
    ↓
Output      result in a register, optionally stored back to memory
```

## 3. Real-World Usage

Every compiled program you have ever run depends on register allocation. **LLVM**, which compiles Rust, Swift and much of the C/C++ world, dedicates an entire optimisation stage to deciding what earns a register — one of the largest single sources of speed in generated code.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A few dozen tiny storage slots inside the CPU |
| **Why does it exist?** | Because execution units need operands in a fraction of a nanosecond |
| **Where does it belong?** | The very top of the memory hierarchy |
| **When should I use it?** | Never directly — but understand it to reason about context switches and compiled code |

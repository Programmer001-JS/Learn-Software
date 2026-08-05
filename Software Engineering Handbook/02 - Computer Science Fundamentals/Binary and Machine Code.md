# Binary and Machine Code

> **In one line —** everything in a computer is numbers, and the numbers a CPU treats as commands are called machine code.

| | |
|---|---|
| **Category** | Foundation Concept |
| **Architectural Layer** | Physical / Instruction |
| **Related notes** | [CPU](CPU.md) · [Registers](Registers.md) · [Compiled vs Interpreted Languages](../03%20-%20Programming%20Languages%20and%20Runtime/Compiled%20vs%20Interpreted%20Languages.md) · [Bytecode](../03%20-%20Programming%20Languages%20and%20Runtime/Bytecode.md) |

---

## 1. Short Definition

*What is it?*

**Binary** is counting with only two digits, 0 and 1, because a transistor is either off or on. **Machine code** is binary that the [CPU](CPU.md) interprets as instructions — the only language a processor actually understands.

---

## 2. Purpose

*What is its main purpose?*

To represent everything — numbers, text, images, sound, programs — in the single physical form a computer can store and manipulate: presence or absence of an electrical charge.

---

## 3. Problem

*What engineering problem does it solve?*

Electronics can reliably distinguish "voltage present" from "voltage absent". Distinguishing ten different voltage levels reliably, at billions of times per second, across a hot chip, is far harder. Binary is the encoding that survives physical reality.

```text
Physical world     →  transistor off / on
    ↓
Binary             →  0 / 1
    ↓
Bytes              →  8 bits, 256 possible values
    ↓
Everything else    →  numbers, text, images, instructions
```

---

## 4. Architecture Position

Machine code is the lowest software layer — the boundary where software becomes electricity.

```text
Python / JavaScript / Java source
    ↓  compiler or interpreter
Bytecode                       (optional intermediate step)
    ↓  JIT / VM
Machine code                   ← you are here
    ↓
CPU instructions
    ↓
Transistors switching
```

---

## 5. Real World Example

- **Every executable file** on your machine — `.exe`, an ELF binary, a compiled Go program — is machine code.
- **Docker images** ship machine code compiled for a specific architecture, which is why an image built for x86-64 will not run on ARM without emulation.
- **Apple's M-series transition** required either recompiling to ARM machine code or translating x86 machine code at install time (Rosetta 2).
- **Character encoding bugs** — a name rendering as `Ivan` instead of `Иван` is a disagreement about how bytes map back to text.

---

## 6. Input

*What does it receive as input?*

For the CPU: a stream of bytes fetched from memory. It does not "know" they are instructions — it treats whatever the instruction pointer points at as an instruction. That is precisely why buffer-overflow attacks were historically able to execute injected data.

---

## 7. Processing

*What happens inside it?*

An instruction encodes an **opcode** (what to do) and its **operands** (what to do it to):

```text
Assembly            Machine code (hex)   Meaning
MOV  eax, 5         B8 05 00 00 00       put the number 5 in register eax
ADD  eax, ebx       01 D8                add ebx to eax
RET                 C3                   return from the function
```

Assembly language is a human-readable naming of exactly these bytes — a one-to-one mapping, not an abstraction.

---

## 8. Output

*What does it return?*

The effect of the instruction: a changed [register](Registers.md), a changed memory location, a changed instruction pointer. There is no other kind of result at this level.

---

## 9. Internal Idea

*How does it work internally?*

Counting in base 2 instead of base 10:

```text
Binary    Decimal
0000        0
0001        1
0010        2
0011        3
1010       10
1111       15
11111111  255      ← one byte, the largest 8-bit value
```

Every layer above this is an agreement about **interpretation**. The same byte `01000001` is the number 65, the letter `A` in ASCII, or part of a machine instruction — depending entirely on what is reading it.

> [!IMPORTANT]
> A file has no inherent type. A `.jpg` is bytes that image software has agreed to interpret as pixels. Type is convention, never a property of the data.

---

## 10. Communication

- **[CPU](CPU.md)** — executes it
- **Compilers and assemblers** — produce it
- **Linkers and loaders** — combine it and place it into memory
- **[Operating system](02%20-%20Operating%20Systems.md)** — loads executables and marks which memory pages may be executed

---

## 11. Dependencies

- An **instruction set architecture** — x86-64, ARM64, RISC-V. Machine code is not portable between them.
- An operating system and executable format (ELF on Linux, PE on Windows, Mach-O on macOS)

---

## 12. Alternatives

```text
Machine code    fastest, tied to one CPU architecture
    ↓
Bytecode        portable, needs a VM  (JVM, .NET, CPython)
    ↓
Interpreted     no compilation step, slowest, most flexible
    ↓
WebAssembly     portable binary format designed for sandboxed execution
```

---

## 13. When To Use

> [!TIP]
> You will almost never write machine code. You need to *understand* it when reading a stack trace, debugging a crash dump, reasoning about why a binary will not run on a different architecture, or thinking about how compilers optimise.

---

## 14. When NOT To Use

> [!CAUTION]
> Writing assembly by hand is essentially never justified in application development. Modern compilers optimise better than humans in almost all cases, and hand-written assembly is unportable, unreadable and unmaintainable.

---

## 15. Advantages

- The fastest possible execution — nothing sits between it and the hardware
- Complete control over the machine
- No runtime, no interpreter, no startup cost

---

## 16. Disadvantages

- Tied to a single CPU architecture
- Essentially unreadable for humans
- No safety net whatsoever — no type checking, no bounds checking
- Impossible to maintain at any scale

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | The baseline against which every other execution model is measured |
| **Memory** | Compact — no interpreter or runtime overhead |
| **CPU** | This *is* what the CPU runs |
| **GPU** | GPUs have their own separate instruction sets |
| **Network** | Not involved |

---

## 18. Security Considerations

> [!CAUTION]
> Because the CPU cannot distinguish data from instructions on its own, historic buffer overflows let attackers write machine code into a data buffer and jump to it. Modern defences — **DEP/NX** (memory marked non-executable), **ASLR** (randomised addresses) and **stack canaries** — all exist to address this one property.

- Compiled binaries can be reverse-engineered; shipping code is not the same as hiding logic
- Supply-chain risk is severe here: a compromised compiler produces compromised machine code, and the source looks perfectly innocent

---

## 19. Mental Model

> [!NOTE]
> **Binary = the alphabet. Machine code = words written in that alphabet that the CPU obeys.**
>
> The same letters can spell a shopping list or an order. The paper does not know the difference — only the reader decides.

---

## 20. Mini Architecture Diagram

```text
Source code
    ↓  compiler
Assembly (human-readable machine code)
    ↓  assembler
Machine code (binary)
    ↓  linker → executable file
    ↓  OS loader → memory
CPU executes
```

---

## 21. Complete Request Flow

From typing code to electricity:

```text
x = 5 + 3
    ↓  parser
abstract syntax tree
    ↓  compiler
MOV eax, 5
ADD eax, 3
    ↓  assembler
B8 05 00 00 00
83 C0 03
    ↓  loaded into RAM by the OS
CPU fetches, decodes, executes
    ↓
transistors switch, a register holds 8
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> Everything a computer holds is numbers; meaning comes entirely from what is agreed to read them — and machine code is the numbers the CPU agrees to obey.

---

## 23. Common Mistakes

- **Believing files have inherent types** — an extension is a hint, not a fact
- **Confusing bytecode with machine code** — bytecode needs a virtual machine; machine code does not
- **Assuming binaries are portable** across CPU architectures
- **Thinking compiled code hides your logic** — decompilers are very good
- **Ignoring character encoding** — the majority of "weird characters" bugs are an encoding mismatch, not corruption

---

## 24. Open Source Technologies

- **GCC**, **LLVM/Clang** — compilers that generate machine code
- **NASM**, **GAS** — assemblers
- **objdump**, **Ghidra**, **radare2** — disassembly and reverse engineering
- **WebAssembly** — a portable binary instruction format for the web
- **RISC-V** — an open instruction set you can actually read the spec for

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Convert your age to binary by hand, then check it.
- [ ] Compile a three-line C program and disassemble it (`objdump -d`). Find the instruction that does the addition.
- [ ] Explain in two sentences why a Docker image built on an M1 Mac may not run on a cloud x86 server.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Source code
    ↓
Bytecode  (optional)
    ↓
Machine code
    ↓
CPU
    ↓
Transistors
```

## 2. Request Flow

```text
Input       source code written by a human
    ↓
Processing  parsed, compiled, assembled, linked, loaded
    ↓
Output      binary instructions the CPU executes directly
```

## 3. Real-World Usage

**Apple's move from Intel to ARM** was fundamentally a machine-code problem: existing applications were binaries compiled for x86-64. Rosetta 2 solved it by translating those instructions to ARM ahead of time — an enormous engineering effort caused entirely by machine code not being portable.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Base-2 representation, and the instruction encoding a CPU executes |
| **Why does it exist?** | Because transistors can only reliably represent two states |
| **Where does it belong?** | The boundary between software and hardware |
| **When should I use it?** | Understand it for debugging and portability; almost never write it |

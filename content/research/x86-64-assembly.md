+++
title = "x86-64 Assembly Registers"
date = 2026-01-26T00:00:00+08:00
description = "Understanding x86-64 assembly registers and how the CPU uses them"
tags = ["lowlevel"]
+++

# Assembly Registers
We already talked about registers earlier when explaining the memory hierarchy, but as a refresher:  
**processor registers are small, volatile storage areas built directly into the CPU**.

They are the fastest storage in the system and are where the CPU actually performs operations.

On x86-64, Intel CPUs have **16 general-purpose registers**, plus the **instruction pointer (RIP)**, which points to the next instruction to execute. Some of these registers are *architecturally* general-purpose but have *conventional* roles.

On x86-32 systems, registers are **32 bits wide**, and there are only **8 general-purpose registers**, plus the instruction pointer.

## Intel Register Evolution
Registers were not replaced over time, they were **extended**. Older registers still exist as sub-registers.

```
+-------------------------------------------------------------+
| Register | Width | Processor / Notes                        |
+-------------------------------------------------------------+
|  A       |  8    | Intel 8008                               |
|  AX      | 16    | Intel 8086 ("A-extended")                |
|  EAX     | 32    | Intel 80386                              |
|  RAX     | 64    | x86-64 (AMD Opteron / Intel Pentium 4)   |
+-------------------------------------------------------------+
```

```
RAX  (64-bit)
└── EAX (32-bit)
    └── AX  (16-bit)
        ├── AL (low 8 bits)
        └── AH (high 8 bits, legacy)
```

Older registers were never removed, they became smaller views into newer, wider registers.

Sub-register access remains available:
- **AL** → low 8 bits  
- **AH** → high 8 bits (legacy, only for AX/BX/CX/DX)

## General-Purpose Registers (x86-64)
```
64-bit   32-bit   16-bit   8-bit

RAX      EAX      AX       AL / AH
RBX      EBX      BX       BL / BH
RCX      ECX      CX       CL / CH
RDX      EDX      DX       DL / DH

```

Disassemblers almost always use the **historical names** (**RAX**, **RBX**, etc.) rather than numeric identifiers, because they are easier to read and match older documentation.
## Additional General-Purpose Registers


```
64-bit   32-bit   16-bit   8-bit     Convention

RSP      ESP      SP       SPL       Stack pointer
RBP      EBP      BP       BPL       Stack base frame
RSI      ESI      SI       SIL       Source index
RDI      EDI      DI       DIL       Destination index

```

- `RSP` is **special by ABI convention** and should always point to the stack.
- Older x86 did not allow byte-level access to `SP`; AMD introduced `SPL` in x86-64 to make naming consistent.
- Although these registers have conventional roles, they are still **general-purpose at the hardware level** and can be repurposed by the compiler.

## Extra Registers Added in x86-64
x86-64 introduced **eight additional registers** to help compilers keep more values in registers:

```
R8  R9  R10 R11
R12 R13 R14 R15
```

Each of these supports full-width access:

```
R8   (64)
R8D  (32)
R8W  (16)
R8B  (8)
```

## Instruction Pointer
```
RIP → points to the next instruction to execute
```

- Not a general-purpose register
- Modified indirectly via **call**, **jmp**, **ret**, etc.

For a quick reference, see this x86-64 register [cheat sheet](https://ost2images.s3.amazonaws.com/Arch101_x86-64_Asm/CheatSheet_x86-64_Registers.pdf).

# Why should we care about accessing smaller parts of registers?
Not all operations use 64-bit values.

Many data types are smaller than 64 bits, and CPUs need a way to operate on those sizes correctly and efficiently. On x86-64, accessing the lower portions of registers (such as 32-bit, 16-bit, or 8-bit parts) is still very important.

Registers can hold **64 bits**, but many data types are smaller:

- `char` → 8 bits  
- `short` → 16 bits  
- `int` → usually 32 bits  

Because of this, the CPU must be able to:
- operate on smaller values  
- store and load smaller values  
- follow the rules of the programming language  

That’s why accessing **parts of a register** still matters.

## Example: 32-bit arithmetic behavior
Consider this C code:

```c
unsigned int x = 1; // On most systems unsigned int is 32 bits 
x += UINT_MAX;      // UINT_MAX is 2^32 - 1
if (x)
    printf("Nope");
````

Mathematically:
```
1 + (2^32 - 1) = 2^32 → wraps to 0
```

If the CPU did this using a full 64-bit register without truncating, the result would be 2^32, not 0, and the program would behave incorrectly.
By doing the operation in a 32-bit register (EAX), the CPU automatically enforces the correct wraparound behavior.

### Why this still matters on x86-64
* Many values in programs are 8, 16, or 32 bits
* Memory loads and stores often use smaller widths
* Function return values often use specific register sizes
* Writing to a 32-bit register automatically clears the upper 32 bits (this is intentional and useful)
* Compilers rely heavily on this behavior

### Is this only for backward compatibility?
Partially.

* **High-byte registers (AH, BH, CH, DH)** exist mainly for backward compatibility and are rarely used in modern code
* **Lower-width access (AL, AX, EAX)** is still essential and heavily used

You will encounter these forms frequently when reading assembly, so understanding them avoids constantly checking manuals.

# Intel Register Conventions
Intel originally suggested certain **usage conventions** for registers in their manuals. These conventions are mostly based on register names and intended roles.
However, these are only **recommendations**, not strict rules. Compilers are free to use registers however they want, and in simple assembly examples, many of these conventions won’t always appear.

### RAX — Accumulator / Return value
- Commonly used to store **function return values**
- Historically, Intel suggested using it as an *accumulator* (e.g., results of arithmetic go back into **AX** / **RAX**)
- Modern compilers don’t strictly follow this accumulator pattern anymore, but **RAX** is still important

### RBX — Base register
- Originally suggested as a pointer to the data section
- Sometimes seen in real-world code, but in simple examples it may not appear much

### RCX — Counter register
- Often used as a **loop counter**
- The name comes from “C” for counter
- Common in loop-related instructions

### RDX — Data / I/O register
- Historically used for I/O-related purposes

### RSI — Source index
- Used as the **source pointer** in string or memory operations

### RDI — Destination index
- Used as the **destination pointer**
- Often paired with `RSI` (source → destination)

Note: Even though registers like `RSI` and `RDI` have traditional roles, they are still **general-purpose registers**. The compiler can use them however it wants.

### RSP — Stack pointer
- Points to the **top of the stack**
- Represents the most recent value pushed onto the stack
- Has special meaning and should not be used arbitrarily

### RBP — Base pointer
- Points to the **base of the current stack frame**
- Commonly used when working with function calls and local variables

### RIP — Instruction pointer
- Points to the **next instruction to execute**
- Not a general-purpose register
- Updated automatically by the CPU

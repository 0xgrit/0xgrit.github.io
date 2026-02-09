+++
title = "Understanding Function Calls in x86-64 Assembly"
date = 2026-01-30
description = "Understanding how function calls work in x86-64 assembly, using a simple C program."
+++


# Wut are we exploring today?
Let's explore how function calls work at the assembly level by examining a simple C program.

## Example C Program

```c
#include <stdio.h>

int func()
{
    return 0xbeef;
}

int main()
{
    func();
    return 0xf00d;
}
```

## Disassembled Output
```asm
main:
140002910  sub     rsp, 0x28
140002914  call    __main
140002919  mov     eax, 0xf00d
14000291e  add     rsp, 0x28
140002922  retn    {__return_addr}
```

To understand this assembly code, we need to learn some new instructions.


## The CALL Instruction
The **CALL** instruction transfers control to a different function in a way that allows execution to resume where it left off. Here's what it does:

1. **Pushes the return address** — The address of the next instruction is pushed onto the stack
2. **Updates RIP** — Changes the instruction pointer (RIP) to the address of the target function

**Syntax:**
```asm
call function
```

The destination address can be specified in multiple ways:
- **Absolute address** — Direct memory address
- **Relative address** — Offset relative to the end of the instruction
- **Register-based** — Address stored in a register

## The RET Instruction
The **RET** instruction returns control from a function. There are two forms:

### Form 1: Simple Return
Pops the top of the stack into RIP (the pop implicitly increments RSP).

**Syntax:**
```asm
ret
```

### Form 2: Return with Stack Cleanup
Pops the top of the stack into RIP and adds a constant number of bytes to RSP.

**Syntax:**
```asm
ret 0x8
ret 0x20
```

You'll commonly see this second form when disassembling Windows APIs.


## Assembly Syntax: Intel vs AT&T

There are two main syntax styles for x86 assembly. The key difference is the **order of operands**.

### Intel Syntax (Windows)
**Format:** `Destination ← Source(s)`

Think of it like assignment in programming: `y = 2x + 1`

```asm
mov rbp, rsp        ; rbp = rsp
add rsp, 0x14       ; rsp = rsp + 0x14
```

### AT&T Syntax (Unix/GNU)
**Format:** `Source(s) → Destination`

Think of it like an equation: `1 + 2 = 3`

```asm
mov %rsp, %rbp      ; rsp → rbp
add $0x14, %rsp     ; 0x14 + rsp → rsp
```

**AT&T Syntax Differences:**
- Registers get a `%` prefix
- Immediates get a `$` prefix


## The MOV Instruction

The `MOV` instruction copies data between locations.

### Valid Move Operations

**Register to Register:**
```asm
mov rbx, rax
```

**Memory to Register:**
```asm
mov rax, [rbx]
mov rax, [rbx+rcx*X]
mov rax, [rbx+rcx*X+Y]
```

**Register to Memory:**
```asm
mov [rbx], rax
mov [rbx+rcx*X], rax
mov [rbx+rcx*X+Y], rax
```

**Immediate to Register:**
```asm
mov rbx, imm64
```

**Immediate to Memory:**
```asm
mov [rbx], imm32
mov [rbx+rcx*X], imm32
mov [rbx+rcx*X+Y], imm32
```

### Important Restriction
**Memory-to-memory moves are NOT allowed.** This is a fundamental x86 architecture limitation.

## Memory Addressing (r/m form)
Memory addresses in x86-64 use a flexible addressing mode called **r/m** (register/memory), which supports complex calculations:

**General Form:**
```
[base + index*scale + displacement]
```

Where:
- **base** = base register (e.g., `rbx`)
- **index** = index register (e.g., `rcx`)
- **scale** = 1, 2, 4, or 8 (represented as X in examples above)
- **displacement** = constant offset (represented as Y in examples above)


## ADD and SUB Instructions
These instructions perform addition and subtraction as you'd expect.

### Operand Rules
- **Destination** can be: r/m or register
- **Source** can be: r/m, register, or immediate
- **Restriction:** Source and destination cannot both be r/m (this would allow memory-to-memory operations, which aren't permitted on x86)

### Examples
```asm
add rsp, 8              ; rsp = rsp + 8
sub rax, [rbx*2]        ; rax = rax - memory[rbx*2]
add [rbx+4], 10         ; memory[rbx+4] = memory[rbx+4] + 10
sub rcx, rdx            ; rcx = rcx - rdx
```


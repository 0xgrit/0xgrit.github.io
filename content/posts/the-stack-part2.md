+++
title = "The Stack Part 2: PUSH, POP, and Stack Operations"
date = 2026-01-28
description = "A detailed look at how PUSH and POP work at the instruction level, including r/m operands, memory addressing modes, and calculating stack offsets"
tags = ["lowlevel"]
+++

# PUSH and POP: The Mechanics
In Part 1, we covered stack frames and how RSP/RBP manage function calls. Now we're diving into the actual instructions that manipulate the stack: **PUSH** and **POP**.

These instructions are simple in concept but have important details worth understanding if you're reading disassembly or writing assembly.

## Quick Refresher

Before we start, here's what you need to remember from Part 1:
- **RSP** points to the top of the stack (lowest address in use)
- **RBP** is the stable base pointer for the current frame
- The stack **grows downward** (toward lower addresses)
- Each 8-byte slot on x86-64 is called a "qword"

If these concepts aren't clear, go read Part 1 first.

# The PUSH Instruction
**PUSH** stores a value on the stack by:
1. Decrementing RSP by 8
2. Writing the value at the new RSP location

```asm
push rbp
```

This is equivalent to:
```asm
sub rsp, 8      ; move stack pointer down
mov [rsp], rbp  ; store value at new top
```

## Why 8 Bytes?

In x86-64:
- Registers are 64 bits = 8 bytes
- The stack is 8-byte aligned
- PUSH always operates on 8-byte values (in 64-bit mode)

## Visual Example

Let's watch `push rax` execute step by step.

**Before:**
```
RAX = 0x0000000000000003
RSP = 0x00007FFF0FE8
```

Stack memory:
```
Address         Value
-----------------------
0x00007FFF0FF0  0x0001
0x00007FFF0FE8  0x0002   ← RSP (current top)
0x00007FFF0FE0  undefined
0x00007FFF0FD8  undefined
```

**Execution:**
```asm
push rax
```

**After:**
```
RAX = 0x0000000000000003  (unchanged)
RSP = 0x00007FFF0FE0      (decreased by 8)
```

Stack memory:
```
Address         Value
-----------------------
0x00007FFF0FF0  0x0001
0x00007FFF0FE8  0x0002
0x00007FFF0FE0  0x0003   ← RSP (new top, contains RAX value)
0x00007FFF0FD8  undefined
```

Notice:
- RSP moved **down** 8 bytes (to a lower address)
- The value from RAX was written to the new stack location
- RAX itself is unchanged

## Why Move RSP First?

RSP must always point to valid stack data. If we wrote the value first and *then* moved RSP, there would be a moment where RSP points to the wrong location. This could break interrupt handlers or debuggers.

By moving RSP first, we ensure it always points to the actual top of the stack.

## PUSH Variants

In 64-bit mode, you can push from two sources:

**1. From a register:**
```asm
push rax        ; push value in rax
push rbx        ; push value in rbx
```

**2. From memory (r/m form):**
```asm
push [rbx]                  ; push value at address in rbx
push [rbx + rcx*4]          ; push from calculated address
push qword [rbx + 0x10]     ; push from rbx + 16
```

Let's break down what "r/m form" actually means.

# Understanding r/m Operands

An **r/m operand** means the instruction can accept either:
- A **register** value, OR
- A **memory** value

That's it. Nothing magical.

## Register vs Memory Syntax

In Intel syntax (what we're using):

```asm
rbx         ; use the VALUE in the register rbx
[rbx]       ; use the value at the MEMORY ADDRESS stored in rbx
```

Square brackets **always** mean memory access. Think of them like pointer dereferencing in C:

```c
x = rbx;     // register value
x = *rbx;    // memory value at address rbx
```

## Why r/m Exists

x86 is flexible. The same instruction can operate on:
- Data in registers (fast)
- Data in memory (slower, but necessary)

Example:
```asm
push rbx        ; push the register value directly
push [rbx]      ; push the value at memory address rbx points to
```

Same instruction, different source.

## The Four Common r/m Addressing Modes

### 1. Register Only
```asm
push rbx
```
Use the value in `rbx`.

### 2. Memory: Base Only
```asm
push [rbx]
```
**Meaning:**
- `rbx` contains a memory address
- Read the value at that address
- Push that value

### 3. Memory: Base + Index × Scale
```asm
push [rbx + rcx*4]
```
**Meaning:**
- Start at the address in `rbx`
- Add `rcx × 4`
- Read the value at the calculated address
- Push it

**Used for:** Array access where `rcx` is the index and 4 is the element size.

### 4. Memory: Base + Index × Scale + Displacement
```asm
push [rbx + rcx*8 + 0x20]
```
**Meaning:**
- Start at `rbx`
- Add `rcx × 8`
- Add constant offset `0x20`
- Read from the final address
- Push it

**Used for:** Accessing fields in arrays of structs.

## Why Only Scale = 1, 2, 4, 8?

These match common data type sizes:
- **1** = byte (`char`)
- **2** = short (`int16_t`)
- **4** = int (`int32_t`)
- **8** = long/pointer (`int64_t`, `void*`)

The CPU can calculate these offsets efficiently in hardware.

## Real-World Example

In C:
```c
int arr[10];
arr[i] = 42;
```

Becomes (roughly):
```asm
mov dword [rbx + rcx*4], 42
```

Where:
- `rbx` = base address of array
- `rcx` = index `i`
- `4` = size of int

## Size Prefixes for Memory Operands

When using memory operands, you must specify the size:

```asm
push qword [rbx]        ; 64-bit (8 bytes)
push dword [rbx]        ; 32-bit (4 bytes) - not valid in 64-bit mode for push
push word [rbx]         ; 16-bit (2 bytes) - not valid in 64-bit mode for push
```

In 64-bit mode, `push` always operates on 8-byte values, so you'll always use `qword` for memory operands.

The assembler needs this because it can't infer the size from the instruction alone when memory is involved.

# The POP Instruction
**POP** retrieves a value from the stack by:
1. Reading the value at RSP
2. Incrementing RSP by 8

```asm
pop rax
```

This is equivalent to:
```asm
mov rax, [rsp]  ; read value from top of stack
add rsp, 8      ; move stack pointer up
```

## Visual Example

Let's watch `pop rax` execute.

**Before:**
```
RAX = 0xAAAAAAAAAAAAAAAA
RSP = 0x00007FFF0FE0
```

Stack memory:
```
Address         Value
-----------------------
0x00007FFF0FF0  0x0001
0x00007FFF0FE8  0x0002
0x00007FFF0FE0  0x0003   ← RSP (current top)
0x00007FFF0FD8  undefined
```

**Execution:**
```asm
pop rax
```

**After:**
```
RAX = 0x0000000000000003  (loaded from stack)
RSP = 0x00007FFF0FE8      (increased by 8)
```

Stack memory:
```
Address         Value
-----------------------
0x00007FFF0FF0  0x0001
0x00007FFF0FE8  0x0002   ← RSP (new top)
0x00007FFF0FE0  0x0003   ; still in memory, but no longer valid
0x00007FFF0FD8  undefined
```

Notice:
- The value `0x0003` still exists at `0x00007FFF0FE0`
- But it's no longer part of the stack
- RSP moved **up** 8 bytes (to a higher address)

## The Old Data Doesn't Disappear
This is important: **POP does not erase memory**.

The value remains in RAM, but it's considered undefined. The next push will overwrite it. Correct programs never read data above the current RSP.

## POP Variants
Like PUSH, POP supports both registers and memory:

**1. Into a register:**
```asm
pop rax         ; load stack value into rax
pop rbx         ; load stack value into rbx
```

**2. Into memory (r/m form):**
```asm
pop [rbx]                   ; store to address in rbx
pop qword [rbx + 0x10]      ; store to rbx + 16
```

Example:
```asm
pop [rbx]
```

**What happens:**
- Read value at `[rsp]`
- Store it to the memory address in `rbx`
- Increment `rsp` by 8

Conceptually:
```
[rbx] = [rsp]
rsp = rsp + 8
```

# The Golden Rule: Balance Your Stack
**Every PUSH must have a matching POP.**

If you don't:
- The stack becomes misaligned
- Return addresses get corrupted
- Your program crashes (usually with a segfault)

Example of correct usage:
```asm
push rax        ; save rax
push rbx        ; save rbx
; ... do work ...
pop rbx         ; restore rbx (note: reverse order!)
pop rax         ; restore rax
```

Example of broken code:
```asm
push rax
push rbx
pop rax         ; WRONG! Should pop rbx first
pop rbx         ; WRONG! This gets the wrong value
```

Remember: **Last In, First Out (LIFO)**.

The last thing you push is the first thing you must pop.

# 32-bit and 16-bit Modes
The same principles apply in other modes, but the sizes change.

## 32-bit Mode

- PUSH/POP operate on 4-byte (32-bit) values
- Uses ESP instead of RSP
- Stack pointer changes by **4** instead of 8

```asm
push eax        ; ESP -= 4, [ESP] = EAX
pop eax         ; EAX = [ESP], ESP += 4
```

## 16-bit Mode
- PUSH/POP operate on 2-byte (16-bit) values
- Uses SP instead of RSP
- Stack pointer changes by **2** instead of 8

```asm
push ax         ; SP -= 2, [SP] = AX
pop ax          ; AX = [SP], SP += 2
```

The principle remains the same—only the operand size and pointer increment differ.

# Calculating Stack Offsets
When reading disassembly, you'll often see code like:

```asm
mov rax, [rbp-0x8]
mov [rbp-0x18], rdi
```

These are accessing local variables on the stack. Let's learn how to calculate these offsets.

## The Basic Question
Given a stack layout, how do you figure out the offset to reach a specific address?

Example:
```plaintext
      HIGH ADDRESSES
            ↑
┌──────────────────────────┐
│ 0xb01dface   ← RBP       │
├──────────────────────────┤
│ 0xaffab1e    ← TARGET    │
├──────────────────────────┤
│ 0x50f7ba11               │
├──────────────────────────┤
│ 0x0000b100d              │
├──────────────────────────┤
│ 0xb100d1e55  ← RSP       │
└──────────────────────────┘
            ↓
      LOW ADDRESSES
```

**Question:** What offset do I use to access `0xaffab1e`?

## Step-by-Step Method
### Step 1: Count the Slots
How many 8-byte slots are between your starting point and the target?

**From RBP to 0xaffab1e:** 1 slot  
**From RSP to 0xaffab1e:** 3 slots

### Step 2: Multiply by 8

Each slot is 8 bytes:
```
Number of slots × 8 = offset in bytes
```

**From RBP:** 1 × 8 = 8 bytes  
**From RSP:** 3 × 8 = 24 bytes

### Step 3: Determine the Sign
Which direction are you moving?

- **UP** (to higher addresses) = **+** (add)
- **DOWN** (to lower addresses) = **-** (subtract)

**From RBP:** Going DOWN → use **-**  
**From RSP:** Going UP → use **+**

RBP is at a HIGHER address than the target, so we subtract. RSP is at a LOWER address, so we add.

### Step 4: Convert to Hexadecimal
```
8 (decimal) = 0x08 (hex)
24 (decimal) = 0x18 (hex)
```

### Step 5: Write the Answer
```
{register}{sign}0x{hex_value}
```

**From RBP:** `rbp-0x08`  
**From RSP:** `rsp+0x18`

So the target is:
- 8 bytes **below** RBP
- 24 bytes **above** RSP

## Quick Reference Table

```
Slots  Bytes  Hexadecimal
─────────────────────────
  1      8      0x08
  2     16      0x10
  3     24      0x18
  4     32      0x20
  5     40      0x28
  6     48      0x30
  7     56      0x38
  8     64      0x40
```

## Practice Example

```plaintext
      HIGH ADDRESSES
            ↑
┌──────────────────────────┐
│ 0xdeadbeef   ← RBP       │
├──────────────────────────┤
│ 0xcafebabe               │
├──────────────────────────┤
│ 0x1badb002   ← TARGET    │
├──────────────────────────┤
│ 0xfeedface               │
├──────────────────────────┤
│ 0x8badf00d   ← RSP       │
└──────────────────────────┘
            ↓
      LOW ADDRESSES
```

**Question:** What is the offset to `0x1badb002`?

**From RBP:**
- Slots: 2 (down from RBP)
- Bytes: 2 × 8 = 16
- Direction: DOWN → negative
- Hex: 16 → 0x10
- **Answer: `rbp-0x10`**

**From RSP:**
- Slots: 2 (up from RSP)
- Bytes: 2 × 8 = 16
- Direction: UP → positive
- Hex: 16 → 0x10
- **Answer: `rsp+0x10`**

## Common Patterns You'll See
**Saving registers before a function call:**
```asm
push rax        ; save caller-saved registers
push rcx
call some_function
pop rcx         ; restore in reverse order
pop rax
```

**Function prologue:**
```asm
push rbp        ; save old frame pointer
mov rbp, rsp    ; establish new frame
sub rsp, 0x20   ; allocate local variables
```

**Function epilogue:**
```asm
leave           ; equivalent to: mov rsp, rbp; pop rbp
ret             ; return to caller
```

bye

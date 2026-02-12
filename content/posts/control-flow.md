+++
title = "Understanding Control Flow in x86-64 assembly"
date = 2026-02-12T00:00:00+08:00
description = "Understanding control flow in x86-64 assembly: how if statements, switch cases, and goto translates to jumps, comparison, and CPU flags."
+++

# Order of Instructions (Control Flow)

Control flow decides which instructions you're going to execute. There are 2 types of control flow:

**Conditional** - where you go somewhere if a condition is met (if statements, switches, loops)
**Unconditional** - where it'll always go somewhere (function calls, goto, exceptions, interrupts)

We already saw in earlier topics that function calls manifest themselves as call/ret. Let's see how goto manifests itself in assembly.

**C code:**
```c
#include <stdio.h>

int main ()
{
  goto mylabel;
  printf ( "skipped!\n" );

mylabel:
  printf ( "mylabel called" );
  return 0xb01dface;
}
```

**Disassembly:**
```asm
main:
    sub rsp, 28h
    jmp 0004012..
    lea rcx, [00..]
    call 000..

$mylabel:
0004012  lea rcx, [00..]
         call 00..
         mov  eax, 0B01DFACEh
         add  rsp, 28h
         ret
```

This isn't hard to understand and is pretty simple. If you observe closely, goto is just **jmp [memory address of the designated label]** - it's literally jumping, as the word itself suggests.

## jmp

Internally, this unconditionally changes RIP to the given address. There are many ways to specify the address:

- **Short relative**: RIP = RIP of next instruction + 1 byte sign-extended to 64 bits displacement. Frequently used in small loops. Some disassemblers will indicate this with the mnemonic "jmp short". For example, `jmp -2` creates an infinite loop. Note that `jmp 00.102` doesn't have the number anywhere in it - it's really `jmp 0x0C` bytes forward. It's not encoded with a 64-bit address baked into it; instead it's saying "2 bytes: one byte to say I'm a jump, and one byte to say I want to jump 0xC bytes forward from the next instruction address."

- **Near relative**: RIP = RIP of next instruction + 4 byte sign-extended to 64 bits displacement

- **Near absolute indirect**: Uses r/m64, which means it could jump to a specific address in a register or pull an address out of memory based on an r/m form

- **Far absolute indirect**: Another addressing form

# if statements (cmp, jne, jle, jge)

```c
int main()
{
    int a = -1, b = 2;

    if ( a == b )
    {
        return 1;
    }

    if ( a > b )
    {
        return 2;
    }

    if ( a < b ) 
    {
        return 3;
    }

    return 0xdefea7;
}
```

```asm
140001540  55                 push    rbp {__saved_rbp}
140001541  4889e5             mov     rbp, rsp {__saved_rbp}
140001544  4883ec30           sub     rsp, 0x30
140001548  e823010000         call    __main
14000154d  c745fcffffffff     mov     dword [rbp-0x4 {var_c}], 0xffffffff
140001554  c745f802000000     mov     dword [rbp-0x8 {var_10}], 0x2
14000155b  8b45fc             mov     eax, dword [rbp-0x4]
14000155e  3b45f8             cmp     eax, dword [rbp-0x8]
140001561  7507               jne     0x14000156a

140001563  b801000000         mov     eax, 0x1
140001568  eb23               jmp     0x14000158d

14000156a  8b45fc             mov     eax, dword [rbp-0x4]
14000156d  3b45f8             cmp     eax, dword [rbp-0x8]
140001570  7e07               jle     0x140001579

140001572  b802000000         mov     eax, 0x2
140001577  eb14               jmp     0x14000158d

140001579  8b45fc             mov     eax, dword [rbp-0x4]
14000157c  3b45f8             cmp     eax, dword [rbp-0x8]
14000157f  7d07               jge     0x140001588

140001581  b803000000         mov     eax, 0x3
140001586  eb05               jmp     0x14000158d

140001588  b8a7fede00         mov     eax, 0xdefea7

14000158d  4883c430           add     rsp, 0x30
140001591  5d                 pop     rbp {__saved_rbp}
140001592  c3                 retn     {__return_addr}
```

We have new instructions:
- **cmp** = compare
- **jne** = jump if not equal
- **jle** = jump if less than or equal
- **jge** = jump if greater than or equal

## jcc (jump if condition is met)

If a condition is true, the jump is taken. Otherwise, it proceeds to the next instruction. There are more than 4 pages of conditional jump types, but many are just synonyms for each other. For example, JNE is equal to JNZ (Jump if Not Equal = Jump if Not Zero; both check if the zero flag ZF == 0).

# What is the Zero Flag?

Let's talk about a special-purpose register: the RFLAGS register. In the manual, EFLAGS is extended to 64 bits and called RFLAGS. The upper 32 bits of RFLAGS register are reserved; the lower 32 bits are EFLAGS. Basically, we just extended the register and aren't really using the extra bits for anything - they're all zeros.

## RFLAGS

The RFLAGS register holds many single-bit flags:

- **Zero Flag (ZF)**: Set to 1 if the result of some instruction is zero; cleared (0) otherwise
- **Sign Flag (SF)**: Set to 1 if the most significant bit (MSB) of the result is 1. For signed values, the sign bit is the MSB. When you divide the range of 8-bit, 32-bit, or 64-bit values into two halves (positive/negative), the MSB is always 1 for negative values
- **Carry Flag (CF)**: Set on unsigned overflow
- **Overflow Flag (OF)**: Set on signed overflow
- **Parity Flag (PF)**: Set if the low byte has an even number of 1 bits
- **Auxiliary Flag (AF)**: Used for BCD arithmetic

# Some Notable JCC Instructions

- **JZ/JE**: Jump if ZF == 1 (Zero/Equal)
- **JNZ/JNE**: Jump if ZF == 0 (Not Zero/Not Equal)
- **JLE/JNG**: Jump if ZF == 1 OR SF != OF (Less or Equal/Not Greater)
- **JGE/JNL**: Jump if SF == OF (Greater or Equal/Not Less)
- **JBE/JNA**: Jump if CF == 1 OR ZF == 1 (Below or Equal/Not Above)
- **JB/JNAE**: Jump if CF == 1 (Below/Not Above or Equal)

No need to memorize this - you'll be running code in a debugger, not just reading it. In the debugger, you can just look at RFLAGS and watch whether it takes a jump.

## Mnemonic Translation

- **A** = Above (unsigned notion) - e.g., if you have 0xFFFFFFFF and you're comparing it to zero, 0xFFFFFFFF is above zero because it's unsigned
- **B** = Below (unsigned notion)
- **G** = Greater than (signed notion) - if it was signed and you were dealing with 0xFFFFFFFF, that would be a negative value and would NOT be greater than zero because it's actually negative
- **L** = Less than (signed notion)
- **E** = Equal (same as Z, zero flag set; sometimes disassemblers will use Z)
- **N** = NOT (e.g., JNL = Jump if Not Less than, JNA = Jump if Not Above)

## Flag Setting

Before you can do a conditional jump, you need something to set the condition status flags for you. This is typically done with:
- **CMP** (compare)
- **TEST** (bitwise AND without storing result)
- Instructions that already have flag-setting side effects (like ADD, SUB, etc.)

## CMP (Compare Two Operands)

The comparison is performed by **subtracting the second operand from the first operand** and then setting the status flags in the same manner as the SUB instruction. 

**What's the difference from just doing SUB?** 

With SUB, the result has to be stored somewhere. With CMP, the result is computed and the flags are set, but **the result is discarded**. It modifies CF, OF, SF, ZF, AF, and PF.

# EZ Guide to Understanding Them All

```asm
cmp 1, 2
```
Is 1 != 2?
```asm
jne wow1    ; IF 1 IS NOT EQUAL TO 2, jump to WOW1
```
This is like: `if (1 != 2);`

```asm
cmp 1, 2
```
Is 1 <= 2?
```asm
jle wow2    ; IF 1 IS LESS THAN OR EQUAL TO 2, jump to WOW2 (signed, because "less" not "below")
```
This is like: `if (1 <= 2);`


```asm
cmp 1, 2
```
Is 1 >= 2?
```asm
jae wow3    ; IF 1 IS GREATER THAN OR EQUAL TO 2, jump to WOW3 (unsigned, because "above" not "greater")
```
This is like: `if (1 >= 2);`

**Note:** Operands are backward in AT&T syntax.

## Takeaways

- Conditional logic like if statements manifests in assembly as conditional jumps: "If condition true, jump there; else fall through"
- Conditions involving (in)equality are often checked with the CMP instruction, which is the same as SUB but throws away the result after the relevant RFLAGS bits are set
- The RFLAGS bits are fundamentally what are checked by the JCC instructions
- On unsigned integers, you'll most likely see **jae** (>=) or **jbe** (<=)
- On signed integers, you'll most likely see **jge** (>=) or **jle** (<=)

# switch Statements 

Switch statements look like a bunch of "if equal" checks. Since if and switch have very similar behavior, it's no wonder they're doing that. I can show you a comparison.

```c
int main()
{
  int a = 1;
  
  switch (a)
  {
    case 0:
      return 0;

    case 1:
      return 1;

    default:
      return 3;
  }

  return 0x32;
}
```

Switch statement when disassembled:
```asm
140001540  55                 push    rbp {__saved_rbp}
140001541  4889e5             mov     rbp, rsp {__saved_rbp}
140001544  4883ec30           sub     rsp, 0x30
140001548  e813010000         call    __main
14000154d  c745fc01000000     mov     dword [rbp-0x4 {var_c}], 0x1
140001554  837dfc00           cmp     dword [rbp-0x4], 0x0
140001558  7507               jne     0x140001561

14000155a  b800000000         mov     eax, 0x0
14000155f  eb1f               jmp     0x140001580

140001561  837dfc01           cmp     dword [rbp-0x4], 0x1
140001565  7507               jne     0x14000156e

140001567  b801000000         mov     eax, 0x1
14000156c  eb12               jmp     0x140001580

14000156e  837dfc03           cmp     dword [rbp-0x4], 0x3
140001572  7507               jne     0x14000157b

140001574  b803000000         mov     eax, 0x3
140001579  eb05               jmp     0x140001580

14000157b  b832000000         mov     eax, 0x32

140001580  4883c430           add     rsp, 0x30
140001584  5d                 pop     rbp {__saved_rbp}
140001585  c3                 retn     {__return_addr}
```

Compare this with equivalent if statements:

```c
int main()
{
  int a = 1;
  
  if (a == 0)
  {
    return 0;
  }

  if (a == 1)
  {
    return 1;
  }

  if (a == 3) 
  {
    return 3;
  }
  else 
  {
    return 0x32;
  }
}
```

When disassembled:
```asm
140001540  55                 push    rbp {__saved_rbp}
140001541  4889e5             mov     rbp, rsp {__saved_rbp}
140001544  4883ec30           sub     rsp, 0x30
140001548  e823010000         call    __main
14000154d  c745fc01000000     mov     dword [rbp-0x4 {var_c}], 0x1
140001554  837dfc00           cmp     dword [rbp-0x4], 0x0
140001558  7507               jne     0x140001561

14000155a  b800000000         mov     eax, 0x0
14000155f  eb1f               jmp     0x140001580

140001561  837dfc01           cmp     dword [rbp-0x4], 0x1
140001565  7507               jne     0x14000156e

140001567  b801000000         mov     eax, 0x1
14000156c  eb12               jmp     0x140001580

14000156e  837dfc03           cmp     dword [rbp-0x4], 0x3
140001572  7507               jne     0x14000157b

140001574  b803000000         mov     eax, 0x3
140001579  eb05               jmp     0x140001580

14000157b  b832000000         mov     eax, 0x32

140001580  4883c430           add     rsp, 0x30
140001584  5d                 pop     rbp {__saved_rbp}
140001585  c3                 retn     {__return_addr}
```

As you can see, they produce nearly identical assembly!


## Additional Section: Signed vs Unsigned Comparisons

The only substantive thing that changes when you use an **unsigned integer** instead of a **signed integer** is the conditional jump instructions that get emitted.

When using unsigned integers, you'll see:
- **JB** (Jump if Below)
- **JA** (Jump if Above)
- **JBE** (Jump if Below or Equal)
- **JAE** (Jump if Above or Equal)

When using signed integers, you'll see:
- **JL** (Jump if Less than)
- **JG** (Jump if Greater than)
- **JLE** (Jump if Less or Equal)
- **JGE** (Jump if Greater or Equal)

**Why does this matter?**

The compiler emits different code depending on whether the programmer declared variables as unsigned versus signed. This means a reverse engineer or decompiler can use these different assembly instructions to infer whether the variables were likely unsigned or signed in the original high-level language.

**How does the hardware handle this?**

The hardware doesn't actually care whether humans interpret bits as signed or unsigned. When executing arithmetic operations like ADD and SUB, the hardware:
1. Performs the operation as if operands were **both** unsigned **and** signed
2. Sets **all** status flags (zero, sign, overflow, carry, parity, etc.)
3. Leaves it to the compiler to emit the correct conditional jump based on whether the high-level code used signed or unsigned types

The compiler figures out what the programmer meant by parsing the high-level language syntax and emits the appropriate signed or unsigned comparison instructions.

The compiler emits different instructions based on whether variables are signed or unsigned, but make sure you step through the assembly yourself to understand what's going on!


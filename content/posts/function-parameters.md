+++
title = "x86-64 Assembly: Function Parameters and Calling Conventions"
date = 2026-02-11T00:00:00+08:00
description = "A walkthrough of how x86-64 functions pass arguments, use the stack, and follow calling conventions."
draft = false
+++

# A silly journey 
Today w'ere going to learn how functions pass parameters, manage the stack, and use the LEA instruction in x86-64 assembly.

## Function Parameters
### Single Parameter Example

Let's start with a simple function that takes one parameter:

```c
int func(int a) {
    int i = a;
    return i;
}

int main() {
    return func(0x11);
}
```

**Disassembly:**

```asm
func:
140001540  push    rbp
140001541  mov     rbp, rsp
140001544  sub     rsp, 0x10
140001548  mov     dword [rbp+0x10], ecx
14000154b  mov     eax, dword [rbp+0x10]
14000154e  mov     dword [rbp-0x4], eax
140001551  mov     eax, dword [rbp-0x4]
140001554  add     rsp, 0x10
140001558  pop     rbp
140001559  retn

main:
14000155a  push    rbp
14000155b  mov     rbp, rsp
14000155e  sub     rsp, 0x20
140001562  call    __main
140001567  mov     ecx, 0x11        ; ← argument in ECX
14000156c  call    func
140001571  add     rsp, 0x20
140001575  pop     rbp
140001576  retn
```

- The argument `0x11` is placed in `ecx` before calling `func`
- `ecx` is a general-purpose register used for the first parameter
- The function moves the parameter through registers and memory (redundant in debug builds)

**Stack Layout:**

```plaintext
Higher addresses ↑
┌─────────────────────────────────────┐
│       main() STACK FRAME            │
│       (starts at 0x14FE00)          │
├─────────────────────────────────────┤
│ 0x14FE08 | return to caller         │ 
│ 0x14FE00 | saved rbp from caller    │ 
│ 0x14FDF8 | 16-byte alignment pad    │
│ 0x14FDF0 | undef                    │
│ 0x14FDE8 | undef                    │
│ 0x14FDE0 | a = ecx = 0x11           │ ← argument!
├─────────────────────────────────────┤
│    main() allocates 0x20 bytes      │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│       func() STACK FRAME            │
│       (starts at 0x14FDD0)          │
├─────────────────────────────────────┤
│ 0x14FDD8 | return to main           │
│ 0x14FDD0 | saved rbp (func's)       │ ← RBP
│ 0x14FDC8 | 16-byte alignment pad    │ 
│ 0x14FDC0 | 16-byte alignment pad    │
│ 0x14FDCC | i = 00000011             │ ← [rbp-0x4]
├─────────────────────────────────────┤ ← RSP
│    func() allocates 0x10 bytes      │
└─────────────────────────────────────┘
Lower addresses ↓
```

### Multiple Parameters Example
Now let's look at a function with 5 parameters:

```c
#define uint64 unsigned long long

int func(uint64 a, uint64 b, uint64 c, uint64 d, uint64 e) {
    int i = a + b - c + d - e;
    return i;
}

int main() {
    return func(0x11, 0x22, 0x33, 0x44, 0x55);
}
```

**Disassembly:**

```asm
main:
140001582  push    rbp
140001583  mov     rbp, rsp
140001586  sub     rsp, 0x30
14000158a  call    __main
14000158f  mov     qword [rsp+0x20], 0x55   ; 5th param on stack
140001598  mov     r9d, 0x44                ; 4th param in r9
14000159e  mov     r8d, 0x33                ; 3rd param in r8
1400015a4  mov     edx, 0x22                ; 2nd param in rdx
1400015a9  mov     ecx, 0x11                ; 1st param in rcx
1400015ae  call    func
```

**Parameter Passing (Microsoft x64):**
- **1st-4th parameters**: `RCX`, `RDX`, `R8`, `R9`
- **5th+ parameters**: Pushed onto the stack

**Stack Layout:**
```plaintext
Higher addresses ↑
┌─────────────────────────────────────┐
│       main() STACK FRAME            │
├─────────────────────────────────────┤
│ 0x14FE08 | return to caller         │
│ 0x14FE00 | saved rbp from caller    │
│ 0x14FDF8 | undef                    │
│ 0x14FDF0 | arg5 = 0x55              │ ← [rbp+0x30] 5th arg (stack)
│ 0x14FDE8 | arg4 = r9 = 0x44         │ ← [rbp+0x28] shadow space
│ 0x14FDE0 | arg3 = r8 = 0x33         │ ← [rbp+0x20] shadow space
│ 0x14FDD8 | arg2 = rdx = 0x22        │ ← [rbp+0x18] shadow space
│ 0x14FDD0 | arg1 = rcx = 0x11        │ ← [rbp+0x10] shadow space
├─────────────────────────────────────┤
│    main() allocates 0x38 bytes      │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│       func() STACK FRAME            │
├─────────────────────────────────────┤
│ 0x14FDC8 | return address to main  │
│ 0x14FDC0 | saved rbp (func's)      │ ← RBP 
│ 0x14FDB8 | 16-byte alignment pad   │ 
│ 0x14FDB0 | i = ffffffef            │ ← [rbp-0x4]
├─────────────────────────────────────┤ ← RSP
│    func() allocates 0x18 bytes      │
└─────────────────────────────────────┘
Lower addresses ↓
```

## Shadow Space
### What is Shadow Space?
From MSDN: The x64 Application Binary Interface (ABI) uses a four-register fast-call calling convention. Space is allocated on the call stack as **shadow space** (or **home space**) for callees to save those registers.

- The caller must always allocate **32 bytes** (4 × 8 bytes) for register parameters
- This is required **even if the function takes fewer than 4 parameters**
- That's why you see `sub rsp, 28h` (40 bytes: 32 shadow + 8 alignment) even for functions with no arguments
- Parameters beyond the first four go on the stack **after** the shadow space

**Why Does It Exist?**
- Provides a consistent calling convention
- Allows callees to spill register parameters to the stack if needed
- Simplifies debugging and stack unwinding

**Example:**
```asm
; Callee can spill register parameters to shadow space
140001548  mov     qword [rbp+0x10], rcx    ; storing rcx in shadow space
14000154c  mov     qword [rbp+0x18], rdx    ; storing rdx in shadow space
140001550  mov     qword [rbp+0x20], r8     ; storing r8 in shadow space
140001554  mov     qword [rbp+0x28], r9     ; storing r9 in shadow space
```

## Calling Conventions
A **calling convention** defines two key aspects:

1. **Register Conventions**: Which registers belong to the caller vs callee
2. **Parameter Passing**: How parameters and return values are passed

Calling conventions are **compiler-dependent**. This guide covers:
- **Microsoft x64 ABI** (Visual Studio)
- **System V x86-64 ABI** (GCC/Linux)


### Caller-Save Registers (Volatile)
**Definition:** The caller assumes these will be **changed by the callee**.

- The **caller** saves them before calling if needed
- The **caller** restores them after the call

**Registers:**
- **Visual Studio**: `RAX`, `RCX`, `RDX`, `R8`, `R9`, `R10`, `R11`
- **GCC**: `RAX`, `RCX`, `RDX`, `R8`, `R9`, `R10`, `R11`, `RDI`, `RSI`


### Callee-Save Registers (Non-Volatile)
**Definition:** The caller assumes these will **NOT be changed by the callee**.

- If the **callee** needs to use them, it must save and restore them
- This preserves the caller's values

**Registers:**
- **Visual Studio**: `RBX`, `RBP`, `RDI`, `RSI`, `R12`-`R15`
- **GCC**: `RBX`, `RBP`, `R12`-`R15`

**Example:**
```asm
; Callee-save example
func:
    push    rbx      ; Save at function entry
    ; ... function body uses rbx ...
    pop     rbx      ; Restore at function exit
    ret
```

### Parameter Passing Conventions
#### Microsoft x64 ABI

**Parameter Registers:**

```plaintext
+----------+----------+
| Position | Register |
|----------|----------|
| 1st      | RCX      |
| 2nd      | RDX      |
| 3rd      | R8       |
| 4th      | R9       |
| 5th+     | Stack    |
+----------+----------+
```

**Return Value:** `RAX` (or `RDX:RAX` for 128-bit)

**Shadow Space:** 32 bytes required

**Example:**
```c
//   RCX    RDX    R8     R9     stack
int func(int a, int b, int c, int d, int e);
```

#### System V x86-64 ABI (GCC)
**Parameter Registers:**

```plaintext
+----------+----------+
| Position | Register |
|----------|----------|
| 1st      | RDI      |
| 2nd      | RSI      |
| 3rd      | RDX      |
| 4th      | RCX      |
| 5th      | R8       |
| 6th      | R9       |
| 7th+     | Stack    |
+----------+----------+
```

**Return Value:** `RAX` (or `RDX:RAX` for 128-bit)
**Shadow Space:** Not required

**Example:**
```c
//   RDI    RSI    RDX    RCX    R8     R9
int func(int a, int b, int c, int d, int e, int f);
```

### Comparison Table
```plaintext
+----------------------+------------------+------------------+
| Feature              | Microsoft x64    | System V AMD64   |
|----------------------|------------------|------------------|
| 1st integer argument | RCX              | RDI              |
| 2nd integer argument | RDX              | RSI              |
| 3rd integer argument | R8               | RDX              |
| 4th integer argument | R9               | RCX              |
| 5th integer argument | stack            | R8               |
| 6th integer argument | stack            | R9               |
| Shadow space         | 32 bytes required| Not required     |
| RSI/RDI status       | Callee-saved     | Caller-saved     |
| Max register args    | 4                | 6                |
+----------------------+------------------+------------------+
```

## 32-bit Calling Conventions
### Common Conventions

**`cdecl`** (default for C):
- **Caller** cleans up the stack
- Parameters pushed right-to-left

**`stdcall`** (Win32 APIs):
- **Callee** cleans up the stack
- Uses special `ret` instruction with stack adjustment
- Parameters pushed right-to-left

### 32-bit Example
```c
int func(int a, int b, int c, int d, int e) {
    int i = a + b - c + d - e;
    return i;
}

int main() {
    return func(0x11, 0x22, 0x33, 0x44, 0x55);
}
```

**Disassembly:**
```asm
main:
00401531  mov     dword [esp+0x10], 0x55   ; 5th arg (e)
00401539  mov     dword [esp+0xc], 0x44    ; 4th arg (d)
00401541  mov     dword [esp+0x8], 0x33    ; 3rd arg (c)
00401549  mov     dword [esp+0x4], 0x22    ; 2nd arg (b)
00401551  mov     dword [esp], 0x11        ; 1st arg (a)
00401558  call    func
```

**Stack Layout:**
```plaintext
Higher addresses ↑
┌─────────────────────────────────────┐
│       main() STACK FRAME            │
├─────────────────────────────────────┤
│ 0x0012FF30 | return to caller       │
│ 0x0012FF2C | saved ebp              │
│ 0x0012FF28 | (local space)          │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│       Arguments (right to left)     │
├─────────────────────────────────────┤
│ 0x0012FF24 | e = 0x55               │ ← [ebp+0x18]
│ 0x0012FF20 | d = 0x44               │ ← [ebp+0x14]
│ 0x0012FF1C | c = 0x33               │ ← [ebp+0x10]
│ 0x0012FF18 | b = 0x22               │ ← [ebp+0x0C]
│ 0x0012FF14 | a = 0x11               │ ← [ebp+0x08]
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│       func() STACK FRAME            │
├─────────────────────────────────────┤
│ 0x0012FF10 | return address         │
│ 0x0012FF0C | saved ebp              │ ← EBP
│ 0x0012FF08 | (unused)               │
│ 0x0012FF04 | (unused)               │
│ 0x0012FF00 | (unused)               │
│ 0x0012FEFC | i = 0xffffffef         │ ← [ebp-0x04]
├─────────────────────────────────────┤ ← ESP
│    func() allocates 0x10 bytes      │
└─────────────────────────────────────┘
Lower addresses ↓
```

**Note:** This is **not** shadow space (64-bit concept). In 32-bit, arguments are simply pushed onto the stack.


### Register Conventions (32-bit)
```plaintext
+--------------+-----+-----+-----+-----+-----+-----+-----+
|              | EAX | EBX | ECX | EDX | ESI | EDI | EBP |
|--------------|-----|-----|-----|-----|-----|-----|-----|
| VS Caller    | ✓   |     | ✓   | ✓   |     |     |     |
| VS Callee    |     | ✓   |     |     | ✓   | ✓   | ✓   |
| SysV Caller  | ✓   |     | ✓   | ✓   | ✓   | ✓   |     |
| SysV Callee  |     | ✓   |     |     |     |     | ✓   |
+--------------+-----+-----+-----+-----+-----+-----+-----+
```

### Stack Frame Linkage (32-bit)
Both `cdecl` and `stdcall` use **explicit stack frame linkage**:

```asm
func:
    push    ebp          ; Save old frame pointer
    mov     ebp, esp     ; Set new frame pointer
    ; ... function body ...
    pop     ebp          ; Restore old frame pointer
    ret

main:
    push    ebp          ; Save old frame pointer
    mov     ebp, esp     ; Set new frame pointer
    call    func
    ; ... rest of main ...
    pop     ebp          ; Restore old frame pointer
    ret
```

- `ebp` points to the base of the current stack frame
- The value at `[ebp]` points to the previous frame's base
- This creates a **linked list** of stack frames
- Parameters: `[ebp + offset]`
- Local variables: `[ebp - offset]`

## Stack Frame Layouts
### 64-bit Stack Frame (Microsoft x64)
```plaintext
                                            Number of entries = 4 or
    Function A                              max parameters
        │                                   (whichever is greater)
        ▼                                         
    ┌─────────────────────┐                      
    │  Stack Parameter    │ ◄───────── Stack parameter area
    │  Stack Parameter    │                      
    │  Stack Parameter    │                      
    │  Stack Parameter    │                      
    ├─────────────────────┤                      
    │      R9 home        │                      
    │      R8 home        │ ◄───────── Register parameter
    │     RDX home        │            shadow space (32 bytes)
    │     RCX home        │                      
    ├─────────────────────┤                      
    │ Caller return addr  │                      
    ├─────────────────────┤───── call B ─────
    │  Local variables    │
    │  Callee-save regs   │
    │   Saved rbp         │ ◄───────── Frame pointer (if used)
    │  _alloca() space    │
    │  Caller-save regs   │
    └─────────────────────┘
        │
        ▼
    Function B (similar structure)
```

### 64-bit Full Stack Diagram
```plaintext
                    stack bottom ↑
    ┌────────────────────────────────────────────┐
    │  return address from main()                │
    ├────────────────────────────────────────────┤
    │  local variables (if any)                  │
    │  Callee-save regs (if any)                 │  main()
 ┄┄>│  Saved rbp (if any)                        │  frame
 ┊  │  _alloca() space (if any)                  │
 ┊  │  Caller-save regs (if any)                 │
 ┊  │  Shadow space + extra args (if any)        │
 ┊  ├────────────────────────────────────────────┤
 ┊  │  return address foo() to main()            │
 ┊  ├────────────────────────────────────────────┤
 ┊  │  local variables (if any)                  │
 ┊  │  Callee-save regs (if any)                 │  foo()
 ┊┄>│  Saved rbp (if any)                        │  frame
 ┊  │  _alloca() space (if any)                  │
 ┊  │  Caller-save regs (if any)                 │
 ┊  │  Shadow space + extra args (if any)        │
 ┊  ├────────────────────────────────────────────┤
 ┊  │  return address bar() to foo()             │
 ┊  ├────────────────────────────────────────────┤
 ┊  │  local variables (if any)                  │
 ┊  │  Callee-save regs (if any)                 │  bar()
 ┊┄>│  Saved rbp (if any)                        │  frame
 ┊  │  _alloca() space (if any)                  │
 ┊  │  ▲ cursor                                  │
    └────────────────────────────────────────────┘
                    stack top ↓
```

**When is a frame pointer used?**
- When using `_alloca()` (stack allocation)
- When required for debugging
- Non-volatile register (usually `rbp`, but could be `r13`)

### System V x86-64 Stack (GCC)
```plaintext
 ┄┄>            stack bottom ↑
 ┊  ┌────────────────────────────────────────────┐
 ┊  │  return address from main()                │
 ┊  ├────────────────────────────────────────────┤
 ┊┄>│  Saved rbp                                 │
 ┊  │  local variables (if any)                  │  main()
 ┊  │  Callee-save regs (if any)                 │  frame
 ┊  │  Caller-save regs (if any)                 │
 ┊  │  Function arguments (if any)               │
 ┊  │  return address foo() to main()            │
 ┊  ├────────────────────────────────────────────┤
 ┊┄>│  Saved rbp                                 │
 ┊  │  local variables (if any)                  │  foo()
 ┊  │  Callee-save regs (if any)                 │  frame
 ┊  │  Caller-save regs (if any)                 │
 ┊  │  Function arguments (if any)               │
 ┊  │  return address bar() to foo()             │
 ┊  ├────────────────────────────────────────────┤
rbp→│  Saved rbp                                 │
 ┊  │  local variables (if any)                  │  bar()
rsp→│  Callee-save regs (if any)                 │  frame
    │  ▲                                         │
    └────────────────────────────────────────────┘
                stack top ↓
```

**Differences:**
- First 6 parameters in registers: `RDI`, `RSI`, `RDX`, `RCX`, `R8`, `R9`
- No shadow space required
- Frame pointers used by default (like 32-bit style)


## LEA Instruction
### What is LEA?

**LEA = Load Effective Address**

The `lea` instruction is an **exception** to the rule that square brackets `[]` mean "dereference memory."

Instead:
1. Calculates a memory address using RMX form
2. Loads that **address** (not the value) into a register

**Analogy:**
```c
int a = 3;
int *pa = &a;  // lea pa, a (gets address, not value)
```

### Example Code

```c
#include <stdlib.h>

int main(int argc, char **argv) {
    int a = atoi(argv[1]);
    return 2 * argc + a;
}
```

**Disassembly:**

```asm
main:
14000156a  mov     eax, dword [rbp+0x10]    ; argc
14000156d  lea     edx, [rax+rax]           ; ← LEA: 2 * argc
140001570  mov     eax, dword [rbp-0x4]     ; a
140001573  add     eax, edx                 ; 2 * argc + a
```

### RMX Addressing Forms
In Intel syntax, square brackets `[]` usually mean "dereference this address."

**RMX Forms:**
1. **Register**: `rbx`
2. **Memory (base)**: `[rbx]`
3. **Memory (base + index × scale)**: `[rbx + rcx * 8]`
4. **Memory (full)**: `[rbx + rcx * 8 + 5]`

Where:
- Scale = 1, 2, 4, or 8
- Displacement = 1 byte (0-255) or 4 bytes (0-2³²)


### LEA Exception
**LEA does NOT dereference memory!**

```asm
lea rax, [rdx + rbx * 8 + 5]
```

**Given:**
- `rbx = 2`
- `rdx = 0x1000`

**Calculation:**
```
rax = 0x1000 + (2 × 8) + 5
    = 0x1000 + 16 + 5
    = 0x1015
```

LEA stores the **calculated value** `0x1015` in `rax`, it does **NOT** read from memory address `0x1015`.

### Common Uses
**1. Pointer Arithmetic**
```asm
lea rax, [rdi + 8]    ; ptr++  (64-bit pointer)
```

**2. Array Element Address**
```asm
lea rax, [rbx + rcx*4 + 8]    ; &array[i] for int array
```

**3. Arithmetic Optimization**
```asm
lea eax, [edi + edi]         ; 2 * edi
lea eax, [edi + edi*2]       ; 3 * edi
lea eax, [edi + edi*4]       ; 5 * edi
lea eax, [edi*8 + edi]       ; 9 * edi
```

The compiler uses `lea` instead of separate `add` and `mul` instructions when it recognizes math that fits the RMX form.

### Special Math Example
From the transcript about `specialmath.c`:

**Key Points:**
1. **Compiler Optimization**: When the compiler sees math that fits the base + index × scale + displacement form, it uses `lea` instead of separate arithmetic instructions

2. **Register Saving**:
```asm
push rbx    ; Callee-save register
; ... function body ...
pop  rbx    ; Restore
```

3. **Stack Alignment**:
- Only `sub rsp, 20h` instead of `28h`
- Because `push rbx` (8 bytes) + return address (8 bytes) = 16 bytes
- Already 16-byte aligned!

### LEA Summary
**Remember:**
- LEA uses square brackets `[]` but does **NOT** dereference
- It calculates an address and stores that value
- Common for pointer arithmetic and optimized math
- Exception to the usual "square brackets = dereference" rule


## Quick Reference
### Identifying Function Arguments
**64-bit MS x64:**
- Count `mov` instructions to `RCX`, `RDX`, `R8`, `R9`
- Check for stack parameters after shadow space

**64-bit System V:**
- Count `mov` instructions to `RDI`, `RSI`, `RDX`, `RCX`, `R8`, `R9`
- Check for stack parameters after 6th argument

**32-bit:**
- Count `push` or `mov [esp+offset]` instructions
- Arguments pushed right-to-left

### Stack Growth
- Higher addresses = stack bottom (older frames)
- Lower addresses = stack top (current frame)
- Stack pointer (`rsp`/`esp`) decreases as stack grows

### Frame Pointer vs Stack Pointer
- **Frame Pointer** (`rbp`/`ebp`): Fixed location in current frame
- **Stack Pointer** (`rsp`/`esp`): Moves with push/pop
- Frame pointer enables fixed offsets for locals and parameters

that's all bye!


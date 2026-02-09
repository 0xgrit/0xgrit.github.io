+++
title = "The Stack Part 1: Stack Frames and Function calls"
date = 2026-01-27
description = "A deep dive into how the stack works in x86-64 by walking through a real C program and its disassembly, explaining stack frames, function calls, and how RSP and RBP are used to manage local variables, arguments, and return flow."
tags = ["lowlevel"]
+++

# Wut is this about?
Today we're going to talk about how the stack works in x86-64 assembly by analyzing a simple C program. We'll examine the disassembled code to understand stack frames, function calls, and the roles of RSP and RBP registers. Yeah I know this shit is hard to understand, that's why we're going to tackle it deeply.

## Sample Program

```c
#include <stdio.h>

void hello(char *name)
{
    int age = 42;
    printf("Hello %s your age is %d\n", name, age);
}

int main(void)
{
    char *name = "Leonardo";
    hello(name);
    return 0;
}
```

## Disassembled Code

### main() Function

```asm
; Function prologue - set up stack frame
0040116d  push    rbp                           ; save old base pointer
0040116e  mov     rbp, rsp                      ; set new base pointer
00401171  sub     rsp, 0x10                     ; allocate 16 bytes for local variables

; Prepare the name variable
00401175  lea     rax, [leonardo_string]        ; load address of "Leonardo" string
0040117c  mov     qword [rbp-0x8], rax          ; store pointer in local variable 'name'

; Call hello() function
00401180  mov     rax, qword [rbp-0x8]          ; load 'name' pointer into rax
00401184  mov     rdi, rax                      ; move to rdi (1st function argument)
00401187  call    hello                         ; call hello(name)

; Function epilogue - clean up and return
0040118c  mov     eax, 0x0                      ; return value = 0
00401191  leave                                 ; restore stack frame
00401192  retn                                  ; return to caller
```

### hello() Function

```asm
; Function prologue - set up stack frame
00401139  push    rbp                           ; save old base pointer
0040113a  mov     rbp, rsp                      ; set new base pointer
0040113d  sub     rsp, 0x20                     ; allocate 32 bytes for local variables

; Store function parameter and initialize local variable
00401141  mov     qword [rbp-0x18], rdi         ; store 'name' parameter (from rdi)
00401145  mov     dword [rbp-0x4], 0x2a         ; age = 42 (0x2a in hex)

; Prepare arguments for printf()
0040114c  mov     edx, dword [rbp-0x4]          ; 3rd arg: age value (42)
0040114f  mov     rax, qword [rbp-0x18]         ; load 'name' pointer
00401153  lea     rcx, [format_string]          ; load format string address
0040115a  mov     rsi, rax                      ; 2nd arg: name pointer
0040115d  mov     rdi, rcx                      ; 1st arg: format string
00401160  mov     eax, 0x0                      ; no floating-point arguments
00401165  call    printf                        ; printf(format_string, name, age)

; Function epilogue - clean up and return
0040116a  nop                                   ; no operation (padding/alignment)
0040116b  leave                                 ; restore stack frame
0040116c  retn                                  ; return to caller
```

### Data Section

```asm
leonardo_string:  "Leonardo"
format_string:    "Hello %s your age is %d\n"
```

## Key Assembly Instructions

Both functions share common patterns. Let's break down the essential instructions:

### Common Instruction Patterns

**Function Prologue (Entry):**
```asm
push rbp          ; Save old base pointer
mov  rbp, rsp     ; Set new base pointer
sub  rsp, N       ; Allocate N bytes for local variables
```

**Function Epilogue (Exit):**
```asm
leave             ; Restore stack frame (equivalent to: mov rsp, rbp; pop rbp)
retn              ; Return to caller
```

### Instruction Reference

#### push
```asm
push source       ; Equivalent to: sub rsp, 8; mov [rsp], source
```
Decrements RSP by 8 bytes, then stores the value at the new stack location.

#### mov
```asm
mov destination, source   ; Copy source to destination
```
Common forms:
- `mov reg, reg` - register to register
- `mov reg, imm` - immediate value to register
- `mov reg, [mem]` - memory to register
- `mov [mem], reg` - register to memory

#### sub
```asm
sub destination, source   ; destination = destination - source
```
Performs subtraction and updates CPU flags (zero, carry, overflow, etc.)

#### lea (Load Effective Address)
```asm
lea rax, [leonardo_string]   ; Load address (not the value) into rax
```
Loads the memory address rather than the value at that address.

#### call
```asm
call target       ; Equivalent to: push rip; jmp target
```
Pushes the return address (next instruction) onto the stack, then jumps to the target.

#### leave
```asm
leave             ; Equivalent to: mov rsp, rbp; pop rbp
```
Restores the previous stack frame.

#### ret/retn
```asm
ret               ; Equivalent to: pop rip; jmp rip
```
Pops the return address from the stack and jumps to it.

## Memory Accesses

### In main()

```asm
00401175  lea     rax, [leonardo_string]        ; Load address of "Leonardo"
0040117c  mov     qword [rbp-0x8], rax          ; Store pointer at [rbp-8]
00401180  mov     rax, qword [rbp-0x8]          ; Read pointer from [rbp-8]
00401184  mov     rdi, rax                      ; Move to rdi (1st argument)
```

**Why `[rbp-0x8]`?**

The notation `[rbp-N]` accesses stack memory relative to the base pointer:
- RBP points to the base of the current stack frame
- Subtracting from RBP accesses local variables within the frame
- Each local variable gets its own offset from RBP

### In hello()

```asm
00401141  mov     qword [rbp-0x18], rdi         ; Store 'name' parameter
00401145  mov     dword [rbp-0x4], 0x2a         ; age = 42
0040114c  mov     edx, dword [rbp-0x4]          ; Load age value
0040114f  mov     rax, qword [rbp-0x18]         ; Load 'name' pointer
```

Notice the pattern: **RBP - N** where N is the offset for each variable.

## Understanding Stack Frames

### What is a Stack Frame?

A **stack frame** is a region of memory within the program's stack that stores information for a single function call. Each function call creates its own stack frame containing:

- Return address (where to return after function completes)
- Saved registers (particularly RBP)
- Function arguments (may be stored here)
- Local variables

### Key Properties

- There is **one stack per thread**
- Each function call uses its own portion (stack frame)
- Frames are created on function entry and destroyed on function exit

### Return Address Storage

When `main()` calls `hello()`:

```asm
00401187  call    hello         ; Pushes return address (0x0040118c) onto stack
0040118c  mov     eax, 0x0      ; This is where execution resumes after hello() returns
```

The processor needs to know where to return after `hello()` completes:

```
Question: "I'm done with hello(), where do I return?"
Answer: "Return to address 0x0040118c (stored in stack frame)"
```

### Local Variables on the Stack

When you declare a local variable:

```c
int age = 42;
```

It's stored in the stack frame:

```asm
00401145  mov     dword [rbp-0x4], 0x2a    ; Store 42 at [rbp-4]
```

This writes the value `0x2a` (42 in hexadecimal) to the stack location `[rbp-0x4]`, which is part of `hello()`'s stack frame.

### Stack Frames with Recursion

Consider a recursive function:

```c
int factorial(int n)
{
    if (n == 0)
        return 1;
    else
        return n * factorial(n - 1);
}
```

Calling `factorial(5)` creates multiple stack frames:

```
┌──────────────────────┐
│  factorial(1) frame  │
├──────────────────────┤
│  factorial(2) frame  │
├──────────────────────┤
│  factorial(3) frame  │
├──────────────────────┤
│  factorial(4) frame  │
├──────────────────────┤
│  factorial(5) frame  │
└──────────────────────┘
```

Each recursive call gets its own stack frame, even though it's the same function.

## The Role of RSP (Stack Pointer)

### Purpose

The **Stack Pointer (RSP)** keeps track of the **top of the stack** - the boundary between used stack memory and available space.

### Stack Growth Direction

In x86-64, the stack **grows downward** toward lower memory addresses.

To extend the stack, we **subtract** from RSP:

```asm
sub rsp, 0x10         ; Allocate 16 bytes (RSP moves to lower address)
```

### Example: Stack Allocation

**Before allocation:**
```
RSP = 0x7fff1000
```

**After `sub rsp, 0x10`:**
```
RSP = 0x7fff0ff0      ; Lower address (stack grew down by 16 bytes)
```

### Visualization

```
High memory (0x7fff1000) ← Old RSP
    |
    | Valid stack memory (allocated space)
    | (This region is fragmented into different stack frames)
    |
    ↓ Stack grows DOWN
Low memory (0x7fff0ff0) ← New RSP (after subtract)
```

### Memory Availability

- **Above RSP (higher addresses)**: Used stack memory
- **Below RSP (lower addresses)**: Unused, available for future allocation

To use more memory, execute: `sub rsp, N` where N is the number of bytes needed.

### Compiler's Role

The compiler calculates how much stack space each function needs based on:
- Local variables
- Function call requirements
- Alignment requirements

This is why you see different values like `sub rsp, 0x10` (16 bytes) in `main()` and `sub rsp, 0x20` (32 bytes) in `hello()`.

## The Role of RBP (Base Pointer)

### Purpose

The **Base Pointer (RBP)** provides a **stable reference point** for accessing the stack frame during function execution.

### Why RBP is Necessary

The stack might grow or shrink during execution depending on different code paths. RSP constantly changes as we push/pop values, but **RBP remains constant** throughout the function's execution.

### Example: Accessing Stack Frame

Before calling `hello()`:

```
       ┌──────────────┐
       │              │
       ├──────────────┤
       │              │
RSP →  ├──────────────┤
       │              │
       │              │ ← main() stack frame
       │              │
       │              │
       │              │
RBP →  ├──────────────┤
       │              │
       └──────────────┘
```

### Step 1: `push rbp`

After `push rbp` in `hello()`, RSP points to the saved RBP value:

```
       ┌──────────────┐
       │              │
RSP →  ├──────────────┤
       │  Saved RBP   │
       ├──────────────┤
       │              │
       │              │ ← main() stack frame
       │              │
       │              │
       │              │
RBP →  ├──────────────┤
       │              │
       └──────────────┘
```

### Step 2: `mov rbp, rsp`

RBP now points to the same location as RSP:

```
       ┌──────────────┐
       │              │
RSP →  ├──────────────┤ ← RBP (new base pointer)
       │  Saved RBP   │
       ├──────────────┤
       │              │
       │              │ ← main() stack frame
       │              │
       │              │
       │              │
       ├──────────────┤
       │              │
       └──────────────┘
```

### Step 3: `sub rsp, 0x20`

Allocate 32 bytes for `hello()`'s stack frame:

```plaintext
RSP →  ┌──────────────┐
       │              │
       │              │
       │              │ ← hello() stack frame
       │  Local vars  │    (local variables, saved args, etc.)
       │              │
       │              │
       ├──────────────┤ ← RBP (constant reference)
       │  Saved RBP   │
       ├──────────────┤
       │              │
       │              │ ← main() stack frame
       │              │
       │              │
       │              │
       ├──────────────┤
       │              │
       └──────────────┘
```

### RBP Stays Constant
Notice that **RBP's position remains fixed** during the entire function execution. Even if we extend the stack further with more `sub rsp` instructions, RBP doesn't move.

This stability makes RBP perfect for accessing stack variables:

```asm
00401141  mov  qword [rbp-0x18], rdi    ; Store at fixed offset from RBP
00401145  mov  dword [rbp-0x4], 0x2a    ; Store at another fixed offset
0040114c  mov  edx, dword [rbp-0x4]     ; Read from fixed offset
0040114f  mov  rax, qword [rbp-0x18]    ; Read from fixed offset
```

### Why Not Use RSP for Access?

If we used RSP to access variables, the offsets would change every time we push/pop values or call other functions. RBP provides a **stable frame of reference**.

## Function Prologue (Detailed)

The **function prologue** sets up the stack frame. Here's what each instruction does:

### Step 1: Save Old Base Pointer

```asm
push rbp              ; Save caller's base pointer
```

**Purpose:** Preserve the caller's RBP so it can be restored later. This is crucial because RBP points to the caller's stack frame, and we need to restore it when we return.

### Step 2: Set New Base Pointer

```asm
mov rbp, rsp          ; RBP now points to current stack top
```

**Purpose:** Establish the base of the new stack frame. After this, RBP becomes our stable reference point for the current function.

### Step 3: Allocate Stack Space

```asm
sub rsp, 0x20         ; Allocate 32 bytes for local variables
```

**Purpose:** Create space for local variables, temporary storage, and alignment requirements.

### Step 4: Use the Stack Frame

```asm
mov qword [rbp-0x8], rax    ; Store data in the stack frame
```

**Purpose:** Now we can safely store and retrieve data using RBP-relative addressing.

## Function Epilogue (Detailed)

The **function epilogue** cleans up the stack frame and returns to the caller.

### Step 1: Restore Stack Frame

```asm
leave                 ; Equivalent to: mov rsp, rbp; pop rbp
```

This single instruction performs two operations:

**First: `mov rsp, rbp`**
- Restores RSP to point at the saved RBP value
- Deallocates all local variables in one instruction

**Second: `pop rbp`**
- Restores the caller's base pointer
- RSP now points to the return address

### Stack Before `leave`:

```
RSP →  ┌──────────────┐
       │  Local vars  │
       │              │
       ├──────────────┤ ← RBP
       │  Saved RBP   │
       ├──────────────┤
       │ Return addr  │
       └──────────────┘
```

### Stack After `leave`:

```
       ┌──────────────┐
       │  (freed)     │
       │              │
       ├──────────────┤
       │  (freed)     │
RBP →  ├──────────────┤ ← Restored to caller's value
       │ Return addr  │ ← RSP
       └──────────────┘
```

### Step 2: Return to Caller

```asm
ret                   ; Pop return address and jump to it
```

**Operation:**
1. Pop the return address from the stack into RIP (instruction pointer)
2. Jump to that address (resume execution in the caller)

### Complete Epilogue Flow

```asm
leave                 ; Clean up stack frame
ret                   ; Return to caller
```

After `ret`, execution continues at the instruction immediately after the `call` in the caller function.

## Ok remember
1. **Stack frames** store function-specific data (return address, local variables, saved registers)
2. **RSP (Stack Pointer)** tracks the top of the stack and changes frequently
3. **RBP (Base Pointer)** provides a stable reference for accessing the current stack frame
4. **Stack grows downward** in x86-64 (toward lower memory addresses)
5. **Function prologue** sets up the stack frame
6. **Function epilogue** cleans up and returns to the caller

### Register Roles
```
┌──────────┬───────────────────────┬─────────────────────────────────────────────────┐
│ Register │ Purpose               │ Behavior                                        │
├──────────┼───────────────────────┼─────────────────────────────────────────────────┤
│ RSP      │ Stack Pointer         │ Points to top of stack; changes with            │
│          │                       │ push/pop/sub/add                                │
├──────────┼───────────────────────┼─────────────────────────────────────────────────┤
│ RBP      │ Base Pointer          │ Points to base of current frame; constant       │
│          │                       │ during function execution                       │
├──────────┼───────────────────────┼─────────────────────────────────────────────────┤
│ RIP      │ Instruction Pointer   │ Points to next instruction; modified by         │
│          │                       │ call/ret/jmp                                    │
└──────────┴───────────────────────┴─────────────────────────────────────────────────┘
```

### Common Patterns

**Function Entry:**
```asm
push rbp              ; Save old base
mov  rbp, rsp         ; Set new base
sub  rsp, N           ; Allocate space
```

**Function Exit:**
```asm
leave                 ; Restore frame
ret                   ; Return to caller
```

**Accessing Local Variables:**
```asm
mov [rbp-offset], value    ; Store variable
mov value, [rbp-offset]    ; Load variable
```

# Uhhh, why did we used different allocation sizes btw
**main() allocates 16 bytes:**
- 8 bytes for `name` pointer
- 8 bytes for alignment (stack must be 16-byte aligned)

Even though main() only needs 8 bytes for data, the compiler rounds up to preserve alignment guarantees.

**hello() allocates 32 bytes:**
- 8 bytes for `name` parameter storage
- 4 bytes for `age` (int)
- 4 bytes padding
- 16 bytes for potential function calls (alignment)

The compiler over-allocates to ensure alignment and leave room for optimizations. That's why

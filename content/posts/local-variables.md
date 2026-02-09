+++
title = "How Local Variables Look in the Stack"
date = 2026-02-09T00:00:00+08:00
description = "A hands-on exploration of how variables are stored in stack memory."
tags = ["lowlevel"]
+++


# GYAT!
Let's explore how the CPU stores variables in stack memory! We'll start simple and build up to more complex examples.

## Example 1: A Simple Integer Variable

Here's our minimal C program:

```c
int func()
{
    int i = 0x5ca1ab1e;
    return i;
}

int main()
{
    return func();
}
```

### Disassembly

```asm
func:
140001710  push    rbp {__saved_rbp}
140001711  mov     rbp, rsp {__saved_rbp}
140001714  sub     rsp, 0x10
140001718  mov     dword [rbp-0x4 {var_c}], 0x5ca1ab1e
14000171f  mov     eax, dword [rbp-0x4]  {0x5ca1ab1e}
140001722  add     rsp, 0x10
140001726  pop     rbp {__saved_rbp}
140001727  retn     {__return_addr}

main:
140001728  push    rbp {__saved_rbp}
140001729  mov     rbp, rsp {__saved_rbp}
14000172c  sub     rsp, 0x20
140001730  call    __main
140001735  call    func
14000173a  add     rsp, 0x20
14000173e  pop     rbp {__saved_rbp}
14000173f  retn     {__return_addr}
```

### Stack Evolution
**After `main()` allocates space (`sub rsp, 0x20`):**

```text
Higher addresses
     |
     +------------------+
     |   main() frame   |
     |   (32 bytes)     |
     +------------------+
     | saved rbp        |
     +------------------+ <- rsp
Lower addresses
```

**After calling `func()` and allocating (`sub rsp, 0x10`):**
```text
     +------------------+
     |   main() frame   |
     |   (32 bytes)     |
     +------------------+
     | saved rbp        |
     | return address   |
     +------------------+
     |   func() frame   |
     |   (16 bytes)     |
     +------------------+
     | saved rbp        |
     +------------------+ <- rsp
```

**After storing variable `i` (`mov dword [rbp-0x4], 0x5ca1ab1e`):**
```text
     +------------------+
     |   func() frame   |
     |   (12 unused)    |
     +------------------+
     | i = 0x5ca1ab1e   | <- [rbp-0x4]
     +------------------+
     | saved rbp        | <- rbp
     +------------------+ <- rsp
```

### Memory View
```text
Address          | Hex Value  | ASCII
-----------------+------------+-------
0x14FDC0         | 5ca1ab1e   | .«¡\
0x14FDC4         | 00000000   | ....
```

**Note:** The value appears as `.«¡\` in ASCII due to **little-endian byte order** — the bytes are stored in reverse as `1e ab a1 5c`.

### Understanding `dword ptr`
`dword ptr` means "4 bytes" (double word). The instruction:

```asm
mov dword ptr [rsp], 5CA1AB1Eh
```

Writes exactly **4 bytes** to memory at the address in `rsp`.

#### Size Qualifiers in Intel Syntax
```asm
mov qword ptr [rsp+0x10], rax   ; 8 bytes (qword)
mov dword ptr [rsp], 0x5C414B1E ; 4 bytes (dword)
mov word  ptr [rsp], ax         ; 2 bytes (word)
mov byte  ptr [rsp], al         ; 1 byte
```

**Key difference:**
- **Registers**: Auto-extend values (zero-extend or sign-extend)
- **Memory**: Writes only the exact number of bytes specified

## Why Does Visual Studio Over-Allocate Space?
We only need 4 bytes for `int i`, but the compiler allocates 16 bytes (`sub rsp, 0x10`). Why?

### Test Case: Multiple Functions

```c
int func3() 
{
    int i = 0x7a11;
    return i;
}

int func2() 
{
    int j = 0x7a1e;
    return func3();
}

int func()
{
    return func2();
}

int main() 
{
    return func();
}
```

### Disassembly

```asm
func3:
140001710  sub     rsp, 0x10
...

func2:
140001728  sub     rsp, 0x30
...

func:
140001742  sub     rsp, 0x20
...

main:
140001755  sub     rsp, 0x20
...
```

### Stack Allocations
```text
+---------+------------+-------+--------------------------------------+
| Function| Allocation | Bytes | Reason                               |
+---------+------------+-------+--------------------------------------+
| func3   | 0x10       | 16    | Minimum alignment                    |
| func2   | 0x30       | 48    | Shadow space + variable + padding    |
| func    | 0x20       | 32    | Shadow space only                    |
| main    | 0x20       | 32    | Shadow space only                    |
+---------+------------+-------+--------------------------------------+
```

### The Rules: x64 Windows Calling Convention

1. **16-byte alignment**: Stack pointer (`rsp`) must always align to 16-byte boundaries
2. **Shadow space**: Functions that call other functions must reserve 32 bytes (0x20) for the callee
3. **Padding**: Extra bytes added to maintain alignment

**Why 16-byte alignment?**
- Required for SSE/AVX instructions (SIMD operations)
- Performance optimization — CPUs access aligned memory faster
- Part of the x64 Windows ABI

### Breaking Down `func2` (0x30 = 48 bytes)

```text
32 bytes (shadow space)
+ 4 bytes (int j)
+ 12 bytes (padding for alignment)
= 48 bytes (0x30)
```

**Notice:** All allocations are multiples of 16: `0x10`, `0x20`, `0x30`!

## Example 2: Two Long Long Variables

```c
int func() 
{
    long long i = 0xf01dab1ef007ba11;
    long long j = 0x0b57ac1e5;
    return i + j;
}

int main() 
{
    return func();
}
```

### Disassembly

```asm
func:
140001710  push    rbp
140001711  mov     rbp, rsp
140001714  sub     rsp, 0x10
140001718  mov     rax, 0xf01dab1ef007ba11
140001722  mov     qword [rbp-0x8], rax
140001726  mov     eax, 0xb57ac1e5
14000172b  mov     qword [rbp-0x10], rax
14000172f  mov     rax, qword [rbp-0x8]
140001733  mov     edx, eax
140001735  mov     rax, qword [rbp-0x10]
140001739  add     eax, edx
14000173b  add     rsp, 0x10
14000173f  pop     rbp
140001740  retn
```

### Stack Layout
**After `sub rsp, 0x10`:**

```text
Higher addresses
     |
     +------------------+
     | return address   | <- 0x14FDD8
     +------------------+
     | saved rbp        | <- 0x14FDD0 (rbp points here)
     +------------------+
     |  (space for i)   | <- 0x14FDC8 [rbp-0x8]
     |    8 bytes       |
     +------------------+
     |  (space for j)   | <- 0x14FDC0 [rbp-0x10]
     |    8 bytes       |
     +------------------+ <- rsp = 0x14FDC0
Lower addresses
```

**After storing both variables:**
```text
     +------------------+
     | return address   |
     +------------------+
     | saved rbp        | <- rbp (0x14FDD0)
     +------------------+
     | i (8 bytes)      | <- [rbp-0x8] (0x14FDC8)
     | 0xf01dab1e       |
     | f007ba11         |
     +------------------+
     | j (8 bytes)      | <- [rbp-0x10] (0x14FDC0)
     | 0x00000000       |
     | 0b57ac1e5        |
     +------------------+ <- rsp
```

### Memory View
```text
Address   | Value      | Variable
----------+------------+---------
0x14FDC0  | 0b57ac1e5  | j (lower 4 bytes)
0x14FDC4  | 00000000   | j (upper 4 bytes)
0x14FDC8  | f007ba11   | i (lower 4 bytes)
0x14FDCC  | f01dab1e   | i (upper 4 bytes)
0x14FDD0  | saved rbp  |
```

**No padding needed!**
- Variable `i`: 8 bytes
- Variable `j`: 8 bytes
- Total: 16 bytes (already a multiple of 16)

## Example 3: Arrays and Type Conversions

```c
short main() 
{
    short a;
    int b[6];
    long long c;
    
    a = 0xbabe;
    c = 0xba1b0ab1edb100d;
    b[1] = a;
    b[4] = b[1] + c;
    
    return b[4];
}
```

### Disassembly

```asm
main:
140001000  sub     rsp, 0x38
140001004  mov     eax, 0xffffbabe
140001009  mov     word [rsp], ax
14000100d  mov     rax, 0xba1b0ab1edb100d
140001017  mov     qword [rsp+0x8], rax
14000101c  mov     eax, 0x4
140001021  imul    rax, rax, 0x1
140001025  movsx   ecx, word [rsp]
140001029  mov     dword [rsp+rax+0x10], ecx
14000102d  mov     eax, 0x4
140001032  imul    rax, rax, 0x1
140001036  movsxd  rax, dword [rsp+rax+0x10]
14000103b  add     rax, qword [rsp+0x8]
140001040  mov     ecx, 0x4
140001045  imul    rcx, rcx, 0x4
140001049  mov     dword [rsp+rcx+0x10], eax
14000104d  mov     eax, 0x4
140001052  imul    rax, rax, 0x4
140001056  movzx   eax, word [rsp+rax+0x10]
14000105b  add     rsp, 0x38
14000105f  ret
```

### Stack Layout

**Initial state (`sub rsp, 0x38` = 56 bytes):**
```text
Higher addresses
     |
rsp->+---------------------+
+0x00|         a           | short (2 bytes)
     +---------------------+
+0x02|     (padding)       | 6 bytes  
     +---------------------+
+0x08|         c           | long long (8 bytes)
     +---------------------+
+0x10|       b[0]          | int (4 bytes)
     +---------------------+
+0x14|       b[1]          | int (4 bytes)
     +---------------------+
+0x18|       b[2]          | int (4 bytes)
     +---------------------+
+0x1c|       b[3]          | int (4 bytes)
     +---------------------+
+0x20|       b[4]          | int (4 bytes)
     +---------------------+
+0x24|       b[5]          | int (4 bytes)
     +---------------------+
+0x28|      (unused)       |
     +---------------------+
+0x38|  (end of alloc)     |
     +---------------------+
Lower addresses
```

**Final state:**
```text
rsp->+---------------------+
+0x00|      0xbabe         | a = 0xbabe (word)
     +---------------------+
+0x02|     (padding)       |
     +---------------------+
+0x08| 0x0ba1b0ab1edb100d  | c (qword)
     +---------------------+
+0x10|      (unused)       | b[0]
     +---------------------+
+0x14|    0xffffbabe       | b[1] = (int)a (sign-extended)
     +---------------------+
+0x18|      (unused)       | b[2]
     +---------------------+
+0x1c|      (unused)       | b[3]
     +---------------------+
+0x20|    0x1ed6abab       | b[4] = b[1] + c (lower 32 bits)
     +---------------------+
+0x24|      (unused)       | b[5]
     +---------------------+
```

### Step-by-Step Execution
**Step 1:** Store `a = 0xbabe`
```asm
mov word [rsp], ax          ; Only lower 16 bits stored
```

**Step 2:** Store `c = 0xba1b0ab1edb100d`
```asm
mov qword [rsp+0x8], rax    ; 8 bytes
```

**Step 3:** `b[1] = a` with sign extension
```asm
mov     eax, 0x4            ; Index 1
imul    rax, rax, 0x1       ; rax = 1 * 4 = 4 (offset)
movsx   ecx, word [rsp]     ; ECX = 0xffffbabe (sign-extended!)
mov     dword [rsp+rax+0x10], ecx  ; Store at rsp+0x14
```

**Why sign-extend?** Because `a` is `short` (signed), and `b` is `int` (signed). The CPU preserves the sign bit.

**Step 4:** `b[4] = b[1] + c`
```asm
mov     eax, 0x4
imul    rax, rax, 0x1       ; rax = 4
movsxd  rax, dword [rsp+rax+0x10]  ; RAX = 0xffffffffffffbabe
add     rax, qword [rsp+0x8]       ; RAX = 0x0ba1b0ab1ed6abab
mov     ecx, 0x4
imul    rcx, rcx, 0x4       ; rcx = 16 (offset for b[4])
mov     dword [rsp+rcx+0x10], eax  ; Store lower 32 bits
```

**Step 5:** Return `b[4]`
```asm
movzx   eax, word [rsp+rax+0x10]  ; EAX = 0xabab (zero-extended)
```

### Memory Map
```text
Offset | Size | Value              | Description
-------+------+--------------------+--------------------------------
+0x00  | 2    | 0xbabe             | a (short)
+0x08  | 8    | 0x0ba1b0ab1edb100d | c (long long)
+0x14  | 4    | 0xffffbabe         | b[1] (sign-extended from a)
+0x20  | 4    | 0x1ed6abab         | b[4] (sum, lower 32 bits only)
```

**Return value:** `0xabab`


## Example 4: Structs in the Stack
Let's take the same three variables from Example 3, but this time we'll put them into a **struct**.

```c
struct MyData 
{
    short a;
    int b[6];
    long long c;
};

short main() 
{
    struct MyData data;
    
    data.a = 0xbabe;
    data.c = 0xba1b0ab1edb100d;
    data.b[1] = data.a;
    data.b[4] = data.b[1] + data.c;
    
    return data.b[4];
}
```

### Disassembly
```asm
main:
140001540  push    rbp {__saved_rbp}
140001541  mov     rbp, rsp {__saved_rbp}
140001544  sub     rsp, 0x50
140001548  call    __main
14000154d  mov     word [rbp-0x2 {var_a}], 0xbabe
140001553  mov     rax, 0xba1b0ab1edb100d
14000155d  mov     qword [rbp-0x10 {var_18}], rax
140001561  movsx   eax, word [rbp-0x2]
140001565  mov     dword [rbp-0x2c {var_34}], eax
140001568  mov     eax, dword [rbp-0x2c]
14000156b  mov     edx, eax
14000156d  mov     rax, qword [rbp-0x10 {var_18}]
140001571  add     eax, edx
140001573  mov     dword [rbp-0x20 {var_28}], eax
140001576  mov     eax, dword [rbp-0x20]
140001579  add     rsp, 0x50
14000157d  pop     rbp {__saved_rbp}
14000157e  retn     {__return_addr}
```

### Stack Layout (Default Alignment)
With default struct packing, the compiler adds padding for alignment:

```text
Higher addresses
     |
rbp->+---------------------+
-0x02|    a = 0xbabe       | short (2 bytes)
     +---------------------+
-0x04|     (padding)       | 2 bytes
     +---------------------+
-0x08|       b[0]          | int (4 bytes)
     +---------------------+
-0x0c|       b[1]          | int (4 bytes)
     +---------------------+
-0x10|       b[2]          | int (4 bytes)
     +---------------------+
-0x14|       b[3]          | int (4 bytes)
     +---------------------+
-0x18|       b[4]          | int (4 bytes)
     +---------------------+
-0x1c|       b[5]          | int (4 bytes)
     +---------------------+
-0x20|     (padding)       | 4 bytes (for alignment)
     +---------------------+
-0x28|    c (8 bytes)      | long long
     |  0x0ba1b0ab1edb100d |
     +---------------------+
-0x30|     (padding)       |
     |        ...          |
     +---------------------+
Lower addresses
```

The struct fields appear in the **same order as declared** (a, b, c), but "upside down" due to stack growth direction.

### Memory Layout in 4-Byte Rows
```text
Offset | Content
-------+-----------------
-0x02  | a = 0xbabe
-0x04  | (padding)
-0x08  | b[0]
-0x0c  | b[1] = 0xffffbabe
-0x10  | b[2]
-0x14  | b[3]
-0x18  | b[4] = 0x1edacacb
-0x1c  | b[5]
-0x20  | (padding)
-0x28  | c = 0x0ba1b0ab1edb100d
```

### Bonus Challenge: `#pragma pack(1)`
What happens if we remove padding with `#pragma pack(1)`?

```c
#pragma pack(push, 1)
struct MyData 
{
    short a;      // 2 bytes
    int b[6];     // 24 bytes
    long long c;  // 8 bytes
};
#pragma pack(pop)
```

### Stack Layout with `#pragma pack(1)`
With `#pragma pack(1)`, all padding is removed and fields are tightly packed:

```text
Higher addresses
     |
rbp->+---------------------+
-0x02|   a = 0xbabe        | 2 bytes (short)
     +---------------------+
-0x06| b[0] = (unassigned) | 4 bytes
     +---------------------+
-0x0a| b[1] = 0xffffbabe   | 4 bytes
     +---------------------+
-0x0e| b[2] = (unassigned) | 4 bytes
     +---------------------+
-0x12| b[3] = (unassigned) | 4 bytes
     +---------------------+
-0x16| b[4] = 0x1edacacb   | 4 bytes
     +---------------------+
-0x1a| b[5] = (unassigned) | 4 bytes
     +---------------------+
-0x22| c = 0x0ba1b0ab1edb100d | 8 bytes (long long)
     +---------------------+
-0x2a|     (padding)       | Visual Studio adds padding
     |        ...          | to reach 16-byte boundary
     +---------------------+
Lower addresses
```

### Memory Layout in 4-Byte Aligned View
When we visualize this in 4-byte rows, things get interesting:

```text
Offset      | Bytes                    | Description
------------+--------------------------+---------------------------
rbp-0x04    | [padding] [padding] 0xba 0xbe | a split across boundary!
rbp-0x08    | [  b[0] - 4 bytes  ]     | b[0] immediately after a
rbp-0x0c    | [  b[1] - 4 bytes  ]     | b[1] = 0xffffbabe
rbp-0x10    | [  b[2] - 4 bytes  ]     | b[2]
rbp-0x14    | [  b[3] - 4 bytes  ]     | b[3]
rbp-0x18    | [  b[4] - 4 bytes  ]     | b[4] = 0x1edacacb
rbp-0x1c    | [  b[5] - 4 bytes  ]     | b[5]
rbp-0x20    | 0x0d 0x10 0xdb 0x1e      | c starts (first 4 bytes)
rbp-0x24    | 0xb1 0x0a 0x1b 0xba      | c continues (last 4 bytes)
rbp-0x28    | [     padding...    ]    | Visual Studio padding
```

**Notice:**
- With `#pragma pack(1)`, the fields are **tightly packed** with no gaps
- `b[0]` starts immediately after `a` (no 2-byte padding)
- `c` starts immediately after `b[5]` (no 4-byte padding)
- However, Visual Studio **still adds padding at the end** to maintain 16-byte alignment for the entire stack frame

### Why the Extra Padding?

Even with `#pragma pack(1)`, Visual Studio allocates the struct in a 16-byte aligned manner:

```text
Struct size without padding: 2 + 24 + 8 = 34 bytes
Next multiple of 16:         48 bytes (0x30)
Padding added:               14 bytes
```

You can test this hypothesis: if you **decrease the size of `b[]`**, you might see different padding behavior!


## Takeaways from Structs
1. **Struct fields are stored in declaration order** (a, b, c)
2. On the stack, they appear "upside down" (lower memory addresses = later fields)
3. **Default alignment** adds padding between fields for performance
4. `#pragma pack(1)` **removes inter-field padding** but may still have end padding
5. Visual Studio maintains **16-byte alignment** for stack frames regardless of packing


## Understanding Key Instructions
### IMUL (Signed Multiply)

Visual Studio prefers `imul` over `mul` (unsigned multiply). There are **3 main forms**:

```asm
imul r/m              ; Single operand:  RDX:RAX = RAX * r/m
imul reg, r/m         ; Two operands:    reg = reg * r/m
imul reg, r/m, imm    ; Three operands:  reg = r/m * imm
```

#### All IMUL Forms

**Group 1 — Single Operand (produces double-width result)**
```asm
imul r/m8   ; AX = AL * r/m8           (8-bit → 16-bit)
imul r/m16  ; DX:AX = AX * r/m16       (16-bit → 32-bit)
imul r/m32  ; EDX:EAX = EAX * r/m32    (32-bit → 64-bit)
imul r/m64  ; RDX:RAX = RAX * r/m64    (64-bit → 128-bit)
```

**Group 2 — Two Operands (may truncate)**
```asm
imul r16, r/m16  ; r16 = r16 * r/m16   (16-bit result)
imul r32, r/m32  ; r32 = r32 * r/m32   (32-bit result)
imul r64, r/m64  ; r64 = r64 * r/m64   (64-bit result)
```

**Group 3 — Three Operands, 8-bit Immediate**
```asm
imul r16, r/m16, imm8  ; r16 = r/m16 * sign-ext(imm8)
imul r32, r/m32, imm8  ; r32 = r/m32 * sign-ext(imm8)
imul r64, r/m64, imm8  ; r64 = r/m64 * sign-ext(imm8)
```

**Group 4 — Three Operands, 16-bit Immediate**
```asm
imul r16, r/m16, imm16  ; r16 = r/m16 * imm16
```

**Group 5 — Three Operands, 32-bit Immediate**
```asm
imul r32, r/m32, imm32  ; r32 = r/m32 * imm32
imul r64, r/m64, imm32  ; r64 = r/m64 * sign-ext(imm32)
```

#### Example 1: Single Operand IMUL
```asm
imul r12b   ; AX = AL * r12b
```

**Before:**
```
r12 = 0x84
rax = 0x609966C1A977E177
```

The instruction multiplies `AL` (0x77) by `r12b` (0x84).

Since this is **signed** multiplication:
- `0x84` as a signed 8-bit value = `-124` (two's complement)
- `0x77` = `+119`
- `-124 × 119 = -14,756` = `0xC65C` (in two's complement)

**After:**
```
rax = 0x609966C1A977C65C
```

Only `AX` changed (lower 16 bits).

#### Example 2: Two Operand IMUL

```asm
imul r12d, eax
```

**Before:**
```
r12 = 0x0000000000000084
rax = 0x609966C1A977E177
```

- `r12d` = `0x84` (positive 132 in 32-bit form)
- `eax` = `0xA977E177` (negative value — MSB is set)

**After:**
```
r12 = 0x0000000061D0415C
```

The result is the lower 32 bits of the product (truncated).

### MOVSX / MOVZX (Move with Extension)
Used to move smaller values into larger registers:

- **MOVZX** (zero-extend): Fills high-order bits with zeros
- **MOVSX** (sign-extend): Fills high-order bits with the sign bit

```asm
movzx rbx, ax   ; Zero-extend AX to RBX
movsx rcx, al   ; Sign-extend AL to RCX
```

### MOVSXD (Move with Sign Extend for Dword)
`MOVSX` only works for 8-bit or 16-bit sources. To sign-extend a 32-bit value to 64 bits:

```asm
movsxd rbx, eax  ; Sign-extend EAX to RBX
```

#### Example
```asm
mov eax, 0xF00DFACE

movzx rbx, eax   ; rbx = 0x00000000F00DFACE (zero-extended)
movsxd rbx, eax  ; rbx = 0xFFFFFFFFF00DFACE (sign-extended)
```

Since `0xF00DFACE` has the MSB set (it's negative in signed interpretation), `movsxd` fills the upper 32 bits with `1`s.

## Takeaways
1. **Stack grows downward** — lower addresses are at the bottom
2. **16-byte alignment** is mandatory on x64 Windows
3. **Shadow space** (32 bytes) is reserved for functions that call others
4. **Little-endian byte order** — bytes are stored in reverse
5. **Local variables** aren't necessarily stored in the order they're declared
6. **Array access** is done via index × element_size arithmetic
7. **Sign extension** preserves the sign bit when converting to larger types
8. **Zero extension** fills with zeros when converting to larger types


## Notes on Memory Before Variable Assignment
The stack memory already contains **garbage values** (leftover data from previous operations, OS setup, or C runtime initialization) because the OS doesn't zero out stack memory when your program starts. This is why you see seemingly random values before variables are assigned.

This is normal and expected behavior!

+++
title = 'Understanding the Computer Memory Hierarchy'
date = 2026-01-23
description = "Understanding the computer memory hierarchy and why computation happens in registers."
tags = ["lowlevel"]
+++

# Computer Memory Hierarchy (RAGHHH)
```
Disk (main.exe)
   ↓  program is loaded
RAM
   ↓  frequently used data is cached
CPU Cache
   ↓  values needed for execution
Registers
   ↓
CPU executes instructions
```

# Okay, so why are we even talking about this?

The memory hierarchy exists to explain **why registers are the only place the CPU actually operates**, and why assembly languages work almost exclusively with registers. Everything else in the system exists mainly to supply data to them.

## wait nah, what are processor registers first?
Processor registers are tiny pieces of memory built directly inside the CPU.

They are:
- the fastest storage in the entire system
- very small in number
- where the CPU actually performs operations

When people say “registers”, they mean processor registers.  
The CPU does not operate directly on RAM or disk; data must first be loaded into registers before any instruction can execute.

For example:
```
c = a + b

load a → register
load b → register
add registers
store result → c
```

If the result is stored (like in **c = a + b**), it is written back to memory.  
If the expression is just **a + b**, the result stays in a register and may never be written to memory.


# So why does the CPU / assembly only operate on processor registers?
So why does the CPU / assembly only operate on processor registers?

Because registers are fast.

If the CPU were to perform arithmetic directly on RAM, execution would be extremely slow. RAM is much slower than registers, and disk-based storage is even slower. The CPU needs data as close and as fast as possible in order to execute instructions efficiently.

## 2 categories of memory
There are also two broad categories of memory: volatile and non-volatile.
- Volatile memory (registers, cache, RAM) loses its contents when power is off.
- Non-volatile memory (SSD, HDD) persists data across power cycles.

It wouldn’t make sense for the CPU to execute instructions directly on non-volatile storage. Those layers exist for persistence, not computation.

## Conceptual Example 
Let’s say we run a program called main.exe.
1. The program lives on disk (main.exe)
2. When we execute it, the OS loads the program into RAM
3. The CPU pulls frequently used instructions and data into cache
4. Before an instruction executes, its operands are loaded into registers
5. The CPU performs the operation using registers

Another example, for example this C code:
```c
int c = a + b;
```

What actually happens conceptually:
```
Disk -> RAM -> Cache -> Registers (add rax, rbx)
```
Even though you wrote C, the CPU only sees register operations.


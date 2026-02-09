+++
title = "What Is NOP?"
date = 2026-01-26T00:00:00+08:00
description = "Understanding the NOP instruction, how it works, and why it is used in assembly"
tags = ["lowlevel"]
+++

# Tf is NOP?
**NOP** stands for **No Operation**.

A NOP instruction does not change registers or memory.  
It simply advances the instruction pointer and consumes execution time.

In other words, the CPU executes it, but nothing observable happens.

## What is NOP used for?
NOP instructions are commonly used for:
- instruction alignment
- padding bytes between instructions

## Why is NOP `0x90`?
On x86, the one-byte NOP instruction (`0x90`) is actually encoded as:

```
XCHG EAX, EAX
```

**XCHG** swaps the values of two registers.  
Swapping a register with itself changes nothing, so this instruction has no effect, making it a perfect NOP.

## Multi-byte NOPs
Although `0x90` is the classic 1-byte NOP, Intel defines **multi-byte NOP instructions** (from 1 to 9 bytes long).

These are used mainly for:
- alignment
- patching code while preserving instruction boundaries

They still perform no operation, just with different instruction lengths.





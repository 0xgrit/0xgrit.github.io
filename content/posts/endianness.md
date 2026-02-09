+++
title = 'Endianness'
date = 2026-01-21T23:59:00+08:00
description = "Explaining little-endian and big-endian systems, and how multi-byte values are stored in memory"
tags = ["lowlevel"]
+++

# What is Endianness?
The term endianness came from Gulliver's Travels where the little end should be cracked first or the big end should be cracked first, and the term is borrowed into computer terminology.

## Little endian 
Little-endian describes a memory layout where the least significant byte of a value is stored at the lowest memory address.

For example, the value **0x123456** stored in memory would appear as:
```
0x56, 0x34, 0x12
```

## Big endian
Big-endian is the opposite layout, where the most significant byte is stored at the lowest memory address.

```
0x12, 0x34, 0x56
```

Network protocols use big-endian ordering, commonly referred to as network byte order.
```
-> man byteorder

DESCRIPTION
       The htonl() function converts the unsigned integer hostlong from host byte order to network byte order.

       On  the i386 the host byte order is Least Significant Byte first, whereas the network byte order, as used on
       the Internet, is Most Significant Byte first.
```

# Reminder
Endianness applies to the ordering of bytes in memory for multi-byte objects. It does not apply to registers or to the ordering of bits within a byte. Why? because registers don't have addressable bytes, memory does.

In C, char is one byte, so endianness does not matter. For multi-byte types like short, endianness determines whether the least significant byte or most significant byte is stored at the lowest memory address. Regardless of endianness, the bits inside each byte keep the same significance (LSB is bit 0, MSB is the highest bit).

Assume a 32-bit value:
```c
uint32_t x = 0xFEEDFACE;
```

Logical value (no memory yet)
```
Most Significant Byte → Least Significant Byte

   MSB                        LSB
┌────────┬────────┬────────┬────────┐
│  0xFE  │  0xED  │  0xFA  │  0xCE  │
└────────┴────────┴────────┴────────┘
```

## BYTE ORDER IN MEMORY
```
┌─────────┬────────────┬────────────────┐
│ Address │ Big-Endian │  Little-Endian │
├─────────┼────────────┼────────────────┤
│  0x03   │    0xCE    │      0xFE      │
│  0x02   │    0xFA    │      0xED      │
│  0x01   │    0xED    │      0xFA      │
│  0x00   │    0xFE    │      0xCE      │
└─────────┴────────────┴────────────────┘
```
Lower memory addresses are shown at the bottom.

```
0xCE = 11001110
0xFA = 11111010
0xED = 11101101
0xFE = 11111110
```

The bits are the same. Only byte position moves.

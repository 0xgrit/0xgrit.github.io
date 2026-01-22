+++
title = 'Decimal, Binary, and Hexadecimal conversion: Part 2'
date = 2026-01-20
description = "Understanding decimal, binary, and hexadecimal conversions (Part 2)"
tags = ["lowlevel"]
+++

# Hey there
Previously, we mainly learned about decimal-to-hexadecimal and binary conversions, and vice versa. However, we lacked depth on binary representations, so today we’ll cover that and also tackle signed and unsigned types in C, including how positive and negative numbers are represented at the bit level.

Alright let's go!

# Converting binary to decimal and hexadecimal
Let's use 0001 0011 0011 0111, as an example. 

### First, group the binary into four!
```
[0001] [0011] [0011] [0111]
```

Then represent the power of 2, from right to left. 
```
 [ 0   0   0   1  ]  [  0   0   1   1  ]  [  0   0   1   1  ]  [  0   1   1   1  ]
   2³  2²  2¹  2⁰       2³  2²  2¹  2⁰       2³  2²  2¹  2⁰       2³  2²  2¹  2⁰ 
```

Now, list out the results of the power of 2 only if the corresponding number is 1 so it'll be
```
 [ 0   0   0   1  ]  [  0   0   1   1  ]  [  0   0   1   1  ]  [  0   1   1   1  ]
   X  X  X  2⁰          X   X  2¹  2⁰       2³  2²  2¹  2⁰       2³  2²  2¹  2⁰ 
        1                       2  1                 2  1            4   2   1                      
```

Then add the numbers on the groups if it's more than 1
```
 [ 0   0   0   1  ]  [  0   0   1   1  ]  [  0   0   1   1  ]  [  0   1   1   1  ]
   X  X  X  2⁰          X   X  2¹  2⁰       2³  2²  2¹  2⁰       2³  2²  2¹  2⁰ 
        1                       (3)                (3)            (7)                     
```

So the decimal representation is 1337, but what about the hexadecimal? You can now map each group directly to its hexadecimal digit.

- 1 = 0x1
- 3 = 0x3
- 3 = 0x3
- 7 = 0x7

Nothing really changes here because the hexadecimal representation of these decimal values is just the same. so let's use another example!

### 0111b
This is a single binary value, so again we represent the powers of 2 from right to left.
```
[  0   1   1   1  ] 
   2³  2²  2¹  2⁰   
```
List out the powers of 2 where the bit is set to 1.
```
[  0   1   1   1  ] 
   X   4   2  1   

```

so 4+2+1 is 7 so that is the decimal representation of 0111, and if you look up the table 7 is just 0x07

# Alright bruh pls sybau, im so overwhelmed in math!!! 
ok calm ur dih down, we're now going to learn about two's complement negative numbers. What does that mean? It involves inverting all bits of the positive number and then adding 1 to obtain its negative representation. but first lets talk about signed and unsigned type in C.

### Signed type
Signed types are data types that allow a numeric variable to represent both positive and negative numbers, as well as zero.
```c
int foo = 1;
```

```cpp
int foo { 1 };
```

what about unsigned types?
### Unsigned types
In contrast, unsigned types can represent zero and positive values only.

```c
unsigned int foo = 1;
```

Both signed and unsigned char have the same number of bits, but for signed char, half of the value range is used to represent negative numbers.
### Signed type visualization
```
[--------] [++++++++]
```

### Unsigned type visualization
```
[++++++++] [++++++++]
```

Let's check this at the bit level so you can see the representation of negative decimal and positive decimal. Let's say you have **0xFF** let's convert this hexadecimal first in decimal form. (Feel free to view the part 1 if you're not familiar with conversions of hexadecimal to decimal form)

F is 15 so we'll represent it as 15.

```
15 15 
```

Then list out the power of 16 from right to left.
```
15   15 
16¹  16⁰
```

so 16 to the power of 1 is 16, and if a number a power of 0 it is 1.
```
so it'll be 
15  15
16  1
```

Now we have to multiply them to the results of the corresponding power of 16
15 x 16 + 15 x 1

Then add them up
240 + 15  = 255

Now lets convert 255 in a binary so we can see how it looks like at the memory level.
```
255 ÷ 2 = 127 remainder 1
127 ÷ 2 = 63  remainder 1
63  ÷ 2 = 31  remainder 1
31  ÷ 2 = 15  remainder 1
15  ÷ 2 = 7   remainder 1
7   ÷ 2 = 3   remainder 1
3   ÷ 2 = 1   remainder 1
1   ÷ 2 = 0   remainder 1
```

Always remember to read the remainders from bottom to top:
```
11111111b
```
The bit representation stays the same, but the interpretation changes. As an unsigned char, 11111111 represents 255. As a signed char, the same bits represent -1.

```c
#include <stdio.h>

int main ()
{
  unsigned char unsignedVar = 0xFF; // stores values 0-255
  signed char signedVar = 0xFF;  // holds both positive and negative numbers

  printf ( "%b\n", unsignedVar );
  printf ( "%b\n", ( unsigned char )signedVar );

  return 0;
}
```

Result when ran:
```
❯ ./main
11111111
11111111
```

Let's try a different number so you can see how two's complement is used to represent a negative value. Let's do 0x0D. 

D is 13, and 0 is 0 so list them out first
```
0 13
```

Now put the power of 16 from right to left
```
0   13
16¹ 16⁰  
```

Now show the value of the corresponding powers
```
0  13
16 1
```

Multiply the decimal representation to the corresponding powers
```
(0 × 16) + (13 × 1)
(0) + (13) = 13
```

Now, lets get the binary representation of 13.
```
13 ÷ 2 = 6  remainder 1
6  ÷ 2 = 3  remainder 0
3  ÷ 2 = 1  remainder 1
1  ÷ 2 = 0  remainder 1
```

Read the remainders from bottom to top:
```
1101
```
and that's the binary representation of 13. Now, what if we want to make it negative? Since **char** is an 8 bits wide, we first pad **1101** with leading 0s.

### Pad 
```
00001101
```

### Flip the bits
```
11110010
```

### Add 1
```
11110011
```

and that's -13 in binary representation.

# Practice question
What do you think is the two's complement representation of the lowest value possible in an 8 byte signed value?

**aaaah nooo what the fuckkk, pls slow down**, don't worry I actually made this blog to make it beginner friendly as much as possible for noobs like me to understand this concept.
Just like in programming, let’s break the problem down into smaller steps.

## What is a two's complement?
Two’s complement is a binary system for representing negative numbers where the negative value is formed by flipping all bits of the positive value and adding 1.

## Second, what is the lowest value possible in an 8 byte signed value?
8 byte is equal to 64 bits, so we can represent it like this

```
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
```

Now that we represented 64 bits, lets talk about MSB. MSB is the most significant bit and it determines if your number is negative(1) or positive(0), if the bit starts with 1, then the value is negative. Since we want to get the lowest value possible in a 64 bit. Our MSB should be one.

And this is our formula, Why -2? because again we are working with binary representations and we're looking for the lowest possible value.
```
N = Bit count
Lowest value = -2^(N-1)
```

Apply it to 64 bits
```
N = 64 bits
Lowest value = -2^(64-1)
Simplified = -2^(63)

This means bit 63 is the leftmost bit
```

Let's represent the lowest value of 64 bits, and use 1 as the MSB because again we're trying to find the lowest value of 64 bits.
```
1000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
```

Note: This value cannot be converted to a positive number using two’s complement within 64 bits, because +2^63 is not representable in a signed 64-bit integer.

### Converting the binary to hexadecimal
Group them into 4 and represent the power of 2 from left to right

```
1   0   0   0 
2³  2²  2¹  2⁰  
```

2 to the power of 3 is 8, so first digit is 8 and all the 15 digits left is zero'd out.
```
0x8000000000000000
```

### Converting the binary to decimal
Since the most significant bit is at position 63, its value is −2^63, which equals **−9223372036854775808** in decimal.

that's all bye!











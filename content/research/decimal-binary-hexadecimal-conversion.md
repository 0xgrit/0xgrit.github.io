+++
title = 'Decimal, Binary, and Hexadecimal conversion'
date = 2026-01-20
description = "Understanding decimal, binary, and hexadecimal conversions"
tags = ["lowlevel"]
+++

# Disclaimer
I'm not an expert. I'm learning alongside you, and everything written here reflects how I currently understand the topic.  
I'm learning by trying to teach others (often called the Feynman method).

Why are you even learning this?
I’m learning x86-64 assembly, so I need to refresh my understanding of numerical systems first. This is just part of a series, and I’ll be documenting everything I learn along the way. If you can’t explain what you’ve learned, then you don’t fully understand it.
  
# The Number Systems
## Decimal
The decimal number system is a **base-10** system.  
This means it uses **10 distinct digits**:

```

0 1 2 3 4 5 6 7 8 9

```

This is the system we use for everyday counting, like `33`, `45`, etc.


## Binary
"Bi" means two, so the binary system is a **base-2** system.

```

0 1

```

- `0` usually represents *off*
- `1` usually represents *on*

Binary is heavily used in computers because hardware naturally maps to two states.


## Hexadecimal
Hexadecimal is a **base-16** number system.

- `hexa` = 6  
- `decimal` = 10  
- `6 + 10 = 16`

First, we reuse the decimal digits:
```

0 1 2 3 4 5 6 7 8 9

```

Then we add six letters to represent values beyond 9:
```

A B C D E F

```

Which correspond to:
- A = 10
- B = 11
- C = 12
- D = 13
- E = 14
- F = 15

So hexadecimal digits represent:
```

0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15

```

That’s **16 values total**, including zero.


# Converting a Decimal Number to Binary
To convert decimal to binary, we use **successive division by 2**.

Rules:
- Divide the number by 2
- Write down the **remainder**
- Use **integer division only**
- Stop when the quotient becomes `0`
- Read the remainders **from bottom to top**


### Example: Convert 15 (decimal) to binary

```

15 ÷ 2 = 7 remainder 1
7  ÷ 2 = 3 remainder 1
3  ÷ 2 = 1 remainder 1
1  ÷ 2 = 0 remainder 1

```

Now read the remainders **bottom to top**:

```

1111b

```

So:
```

15 = 1111b

```

### Another Example: Convert 12 (decimal) to binary
```

12 ÷ 2 = 6 remainder 0
6  ÷ 2 = 3 remainder 0
3  ÷ 2 = 1 remainder 1
1  ÷ 2 = 0 remainder 1

```

Read bottom to top:

```

1100b

```

So:
```

12 = 1100b

```

# Converting a Decimal Number to Hexadecimal
The method is the same as binary conversion, but instead of dividing by 2, we **divide by 16**.


### Example: Convert 15 (decimal) to hexadecimal

```

15 ÷ 16 = 0 remainder 15

```

Since the quotient is `0`, we stop.

Now convert the remainder:
- `15` corresponds to `F` in hexadecimal

So the result is:
```

0x0F

```

### Key Idea
If the number is **smaller than the base**, the quotient will be `0`, and the remainder will be the number itself.

### Another example: Convert 13 (decimal) to hexadecimal
Since 13 is smaller than the base (16), the quotient will be `0` and the remainder will be the number itself.

```

13 ÷ 16 = 0 remainder 13

```

In hexadecimal:
- **13** corresponds to **D**

So the result is: **0x0D**

# Converting a Hexadecimal Number to Decimal
### Example 1: Convert 0x100 to decimal

Write out the hexadecimal digits:
```

1 0 0

```

List out the powers of 16 from left to right:
```

1     0     0
16²   16¹   16⁰

```

So it will be:
```

1 × 16² + 0 × 16¹ + 0 × 16⁰

```

List out the corresponding powers first:
```

1 × 256 + 0 × 16 + 0 × 1

```

(Any non-zero number raised to the power of 0 is equal to 1.)

Simplify it and add:
```

256 + 0 + 0

```

Meaning:
```

0x100 is 256 in decimal.

```

### Example 2: Convert 0x1337 to decimal
Write out the hexadecimal digits:
```
1 3 3 7
```

List out the powers of 16 from left to right:
```
1    3    3    7
16³  16²  16¹  16⁰

```

So it will be:
```
1 × 16³ + 3 × 16² + 3 × 16¹ + 7 × 16⁰
```

Solve it until everything becomes single numbers, step by step so you won’t get confused:
```
4096 + 3 × 256 + 3 × 16 + 7
```

Last step:
```
4096 + 768 + 48 + 7
```

Which is:
```
4919
```

# Converting Hexadecimal to Binary

Let's use 0x0D as an example.

## Convert letters to numbers first if there are any
D corresponds to 13.

Now use this as a base (4-bit values):
```
8 4 2 1
```

What numbers add up to 13?
8, 4, and 1
```
8 4 2 1 
1 1 0 1
```

Even if a digit is 0, we still apply the same principle and always write 4 bits.

### Let's try 0xDF
D = 13, F = 15

For D(13):
```
8 4 2 1 
1 1 0 1
```

because 8 + 4 + 1 is 13 .

For F(15):
```
8 4 2 1
1 1 1 1 
```

Because 8 + 4 + 2 + 1 is 15.

So 0xDF = **11011111**

## Last practice: 0x1337
No letter, so we use the digits directly.

```
8 4 2 1 
0 0 0 1
```

For 3:
```
8 4 2 1 
0 0 1 1 
```

For 3 again:
```
8 4 2 1 
0 0 1 1 
```

For 7 
```
8 4 2 1 
0 1 1 1
```


Putting everything together: 0001 0011 0011 0111
so the binary reperesentation of hexaddecimal 1337 is: **0001001100110111**



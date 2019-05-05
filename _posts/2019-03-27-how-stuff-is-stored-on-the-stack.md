---
layout: post
title: How stuff is stored on the stack
date: 2019-03-27
tags: [c, programming]
---


The stack grows from high addresses to low.

```
memory:

              high addresses
     ===========
     | stack   |
     |---------|
     | |  |  | |
     | v  v  v |
     |         |
     |         |
     | ^  ^  ^ |
     | |  |  | |
     |---------|
     | heap    |
     ===========
              low addresses
```

> Note: Adding an item to the stack is called *pushing*, while removing an item is *popping*.

## Local variables

When we declare local variables, they are pushed onto the stack in the order they're declared:

```c
/*
 * s1.c
 * gcc s1.c -o s1 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    int8_t  a = 1;
    int8_t  b = 2;
    int64_t c = 3;
    int16_t d = 4;
    int32_t e = 150000;

    printf("a: %p | %" PRId8  "\n", (void *)&a, a);
    printf("b: %p | %" PRId8  "\n", (void *)&b, b);
    printf("c: %p | %" PRId64 "\n", (void *)&c, c);
    printf("d: %p | %" PRId16 "\n", (void *)&d, d);
    printf("e: %p | %" PRId32 "\n", (void *)&e, e);

}
```

<div class="term">
<b>~]$</b> ./s1
a: 0x7fff77fc3caf | 1
b: 0x7fff77fc3cae | 2
c: 0x7fff77fc3ca0 | 3
d: 0x7fff77fc3c9e | 4
e: 0x7fff77fc3c98 | 150000
</div>

> **Warning:** `%p` expects an argument of type `void *`, hence the cast.

Notice that *a*, the first variable, has the highest address, while *e*, the last variable, has the lowest.

A compiler typically chooses the most efficient way to store the variables on the stack. This may involve taking care of alignment requirements so that processors have fast access to the data, so there may be padding between variables.

### Assembly

If we look at the assembly code output of `gcc -S e1.c`, we observe the following:

```assembly
    ...
subq    $32, %rsp
movb    $1, -1(%rbp)
movb    $2, -2(%rbp)
movq    $3, -16(%rbp)
movw    $4, -18(%rbp)
movl    $150000, -24(%rbp)
    ...
```

The stack pointer `rsp` (which points to the top of the stack - the lowest address) is decreased by 32, then the variables are pushed onto the stack.

If we examine the executable in a debugger, we can get a really good look behind the scenes:

<div class="term">
<b>~]$</b> gcc -g s1.c
<b>~]$</b> gdb a.out
<b>(gdb)</b> break 16
<b>(gdb)</b> run
</div>

The stack pointer points to the top of the stack (the lowest address):

<div class="term">
<b>(gdb)</b> print $sp
$1 = (void *) 0x7fffffffe270
</div>

The base pointer points to the bottom of the stack:

<div class="term">
<b>(gdb)</b> print %rbp
$2 = (void *) 0x7fffffffe290
</div>

> Note: The difference between the hexadecimal values 290 and 270 is 32 in decimal.

```
memory:

              high addresses
     ===========
     | stack   |
     |---------| <- base pointer
     |---------|
     |---------|
     |---------|
     |---------| <- stack pointer
     | |  |  | |
     | v  v  v |
     |         |
     |         |
     | ^  ^  ^ |
     | |  |  | |
     |---------|
     | heap    |
     ===========
              low addresses
```

All our variables are between the base pointer and the stack pointer:

<div class="term">
<b>(gdb)</b> x/32xb $rsp
0x7fffffffe270: 0xe0    0x05    0x40    0x00    0x00    0x00    0x00    0x00
0x7fffffffe278: 0xf0    0x49    0x02    0x00    0x00    0x00    0x04    0x00
0x7fffffffe280: 0x03    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7fffffffe288: 0x00    0x00    0x00    0x00    0x00    0x00    0x02    0x01
</div>

Starting from the bottom right and moving left, it is easy to spot the values of *a* to *d*. The value of *e* is not easily observable, but is fairly straightforward to decode if we take byte order into consideration.


### Byte order

On a Little-endian platform, the the most significant byte of a number - the octet containing the bit with the highest value bit poistion (the most significant bit) - is stored at a higher address than the least significant byte.

> Note: most significant byte (MSB), least significant byte (LSB).

The address of a variable points to the first byte of the variable (the LSB).

<div class="term">
<b>(gdb)</b> print &e
$13 = (int32_t *) 0x7fffffffe278
</div>

The e varible is 32 bits, 4 bytes, and if we examine these addresses we can recognize them in the above output, on the second line going towards the right.

<div class="term">
<b>(gdb)</b> x/xb 0x7fffffffe278
0x7fffffffe278: 0xf0
<b>(gdb)</b> x/xb 0x7fffffffe279
0x7fffffffe279: 0x49
<b>(gdb)</b> x/xb 0x7fffffffe27a
0x7fffffffe27a: 0x02
<b>(gdb)</b> x/xb 0x7fffffffe27b
0x7fffffffe27b: 0x00
</div>

> Note: We could get the same output using `x/4xb`.

Since the stack grows from high to low addresses, that means a 32-bit number is stored in memory like this:

```
memory:

              high addresses
     ===========
     | stack   |
     |---------|
     |   MSB   | 0x7fffffffe27b
     |---------|
     |   ...   |
     |---------|
     |   ...   |
     |---------|
     |   LSB   | 0x7fffffffe278
     |---------|
     | |  |  | |
     | v  v  v |
     |         |
     |         |
     | ^  ^  ^ |
     | |  |  | |
     |---------|
     | heap    |
     ===========
              low addresses
```

If we examine the address of *e* as a word of bits, it is displayed in big endian order:

<div class="term">
<b>(gdb)</b> x/tw &e
0x7fffffffe278: 00000000000000100100100111110000
</div>

This above bits are easily converted to 150000.

## Floating point

Not surprisingly, floating-point varaibles are also (like all local variables) put on the stack in the order they are declared:

```c
/*
 * s2.c
 * gcc s2.c -o s2 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    float a = 11.15f;
    double b = 1.0;

    printf("a: %p | %f\n", (void *)&a, a);
    printf("b: %p | %lf\n", (void *)&b, b);
}
```

<div class="term">
<b>~]$</b> ./s2
a: 0x7ffd7c2c509c | 11.150000
b: 0x7ffd7c2c5090 | 1.000000
</div>

Let's have a look at how they are stored on the stack:

<div class="term">
<b>~]$</b> gcc -g s2.c
<b>~]$</b> gdb a.out
<b>(gdb)</b> break 13
<b>(gdb)</b> run
<b>(gdb)</b> x/4tb &a
0x7fffffffe28c: 01100110        01100110        00110010        01000001
<b>(gdb)</b> x/8tb &b
0x7fffffffe280: 00000000        00000000        00000000        00000000 \
                00000000        00000000        11110000        00111111
</div>

Floating-point numbers are also stored with the MSB at a higher address than the LSB, which can be easily verfied by converting.

## Arrays

Arrays are collections of elements of a single type stored in a contiguous region of memory with no padding inbetween.

```c
/*
 * s3.c
 * gcc s3.c -o s3 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

#define arrlen(x) (sizeof(x) / sizeof(*x))

int main()
{
    int32_t arr[] = {1, 100, 35000, -100, 6};

    for (int i = 0; i < arrlen(arr); i++)
        printf("arr[%d]: %p | %" PRId32 "\n", i, (void *)&arr[i], arr[i]);
}
```

<div class="term">
<b>~]$</b> ./s3
arr[0]: 0x7ffcab88c5f0 | 1
arr[1]: 0x7ffcab88c5f4 | 100
arr[2]: 0x7ffcab88c5f8 | 35000
arr[3]: 0x7ffcab88c5fc | -100
arr[4]: 0x7ffcab88c600 | 6
</div>

The first element of the array has the lowest address, and the last element has the highest, so the stack looks like this:

```
memory:

              high addresses
     ===========
     | stack   |
     |---------|
     |  arr[4] |
     |---------|
     |  arr[3] |
     |---------|
     |  arr[2] |
     |---------|
     |  arr[1] |
     |---------|
     |  arr[0] |
     |---------|
     | |  |  | |
     | v  v  v |
     |         |
     |         |
     | ^  ^  ^ |
     | |  |  | |
     |---------|
     | heap    |
     ===========
              low addresses
```

Since the first element is at the lowest address, and there is no padding between elements, we can use indexes (`arr[i]`) or offsets (`*(arr + i)`) to read or write specific elements of the array.


### Assembly

Using `gcc -S`, we can see how the array is set up:

```assembly
subq    $32, %rsp
movl    $1, -32(%rbp)
movl    $100, -28(%rbp)
movl    $35000, -24(%rbp)
movl    $-100, -20(%rbp)
movl    $6, -16(%rbp)
```

### Debugger

We can examine the array using a debugger:

<div class="term">
<b>~]$</b> gcc -g s3.c -std=c99
<b>~]$</b> gdb a.out
<b>(gdb)</b> break 14
<b>(gdb)</b> run
<b>(gdb)</b> print &arr
$1 = (int32_t (*)[5]) 0x7fffffffe270
<b>(gdb)</b> x/5dw &arr
0x7fffffffe270: 1       100     35000   -100
0x7fffffffe280: 6
</div>

The individual elements are of course stored in Little-endian format, as we can see when we examine the third element (35000):

<div class="term">
<b>(gdb)</b> x/4tb &arr[2]
0x7fffffffe278: 10111000        10001000        00000000        00000000
</div>

00000000 00000000 10001000 10111000 is 35000.

## Strings

```c
/*
 * s4.c
 * gcc s4.c -o s4 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    uint8_t *str1 =  "Hello, 1!";
    uint8_t str2[] = "Hello, 2!";

    printf("str1: %p | %s\n", (void *)&str1, str1);
    printf("str2: %p | %s\n", (void *)str2, str2);
}
```

<div class="term">
<b>~]$</b> ./s4
str1: 0x7ffc8a212138 | Hello, 1!
str2: 0x7ffc8a212120 | Hello, 2!
</div>

Again, notice that the variables are put on the stack in the order they are declared.

### Debugger

Using a debugger, let's see how strings are stored in memory:

<div class="term">
<b>(gdb)</b> break 13
<b>(gdb)</b> run
<b>(gdb)</b> print $rbp - $rsp
$1 = 32
<b>(gdb)</b> x/32cb $rsp
0x7fffffffe270: 72 'H'  101 'e' 108 'l' 108 'l' 111 'o' 44 ','  32 ' '  50 '2'
0x7fffffffe278: 33 '!'  0 '\000'        64 '@'  0 '\000'        0 '\000'        0 '\000'        0 '\000'        0 '\000'
0x7fffffffe280: 112 'p' -29 '\343'      -1 '\377'       -1 '\377'       -1 '\377'       127 '\177'      0 '\000'        0 '\000'
0x7fffffffe288: 16 '\020'       6 '\006'        64 '@'  0 '\000'        0 '\000'        0 '\000'        0 '\000'        0 '\000'
</div>

When we look at all the bytes in memory between the stack and base pointer, we can easily spot str2:

```
memory:

              high addresses
     ===========
     | stack   |
     |---------|
     |  '\0'   | 0x7fffffffe279
     |---------|
     |   '!'   |
     |---------|
     |   '2'   |
     |---------|
     |   ...   |
     |---------|
     |   'e'   |
     |---------|
     |   'H'   | 0x7fffffffe270
     |---------|
     | |  |  | |
     | v  v  v |
     |         |
     |         |
     | ^  ^  ^ |
     | |  |  | |
     |---------|
     | heap    |
     ===========
              low addresses
```

But the characters of str1 are not seen. That's because str1 is a pointer to a string. The memory location of str1 on the stack contains the address to the memory where the actual string is stored:

<div class="term">
<b>(gdb)</b> x/xw &str1
0x7fffffffe288: 0x00400610
<b>(gdb)</b> x/50cb 0x00400610
0x400610:       72 'H'  101 'e' 108 'l' 108 'l' 111 'o' 44 ','  32 ' '  49 '1'
0x400618:       33 '!'  0 '\000'        115 's' 116 't' 114 'r' 49 '1'  58 ':'  32 ' '
0x400620:       37 '%'  112 'p' 32 ' '  124 '|' 32 ' '  37 '%'  115 's' 10 '\n'
0x400628:       0 '\000'        115 's' 116 't' 114 'r' 50 '2'  58 ':'  32 ' '  37 '%'
0x400630:       112 'p' 32 ' '  124 '|' 32 ' '  37 '%'  115 's' 10 '\n' 0 '\000'
0x400638:       1 '\001'        27 '\033'       3 '\003'        59 ';'  52 '4'  0 '\000'        0 '\000'        0 '\000'
0x400640:       5 '\005'        0 '\000'
</div>

In the output above, we see the characters for str1 as well as the string literals we supplied to printf.

> Note: We could more simply do `print str1` in gdb to view type, memory address of the string, and the actual string.

### Assembly

In the assembly code, we observe the following:

```assembly
        ...

        .section        .rodata
.LC0:
        .string "Hello, 1!"
.LC1:
        .string "str1: %p | %s\n"
.LC2:
        .string "str2: %p | %s\n"


        ...

        subq    $32, %rsp
        movq    $.LC0, -8(%rbp)
        movabsq $3611935758223172936, %rax
        movq    %rax, -32(%rbp)
        movw    $33, -24(%rbp)

        ...
```

The address of the constant string (`.LC0`) is moved to the stack.
A 64-bit decimal value is put in %rax, then pushed to the stack. Then a 16-bit decimal number is pushed to the stack.

The 64-bit decimal number can be decoded as following:

```c
/*
 * s5.c
 * gcc s5.c -o s5 -std=c99
 */
#include <stdio.h>
#include <inttypes.h>

int main()
{
    uint64_t v = 3611935758223172936;
    uint8_t *vp = (uint8_t *)&v;

    for(int i = 0; i < 8; i++)
        printf("%" PRIu8 "\t:\t%c\n", *(vp + i), *(vp + i));
}
```

<div class="term">
<b>~]$</b> ./s5
72      :       H
101     :       e
108     :       l
108     :       l
111     :       o
44      :       ,
32      :
50      :       2
</div>

The 16-bit decimal number (in Little-endian) is 01100101 00000000 which translates to '!' followed by '\0'.

## Structures

Structures are padded, if necessary for alignment:

```c
/*
 * s6.c
 * gcc s6.c -o s6 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    struct st {
        int32_t a;
        int64_t b;
        int8_t  c;
    };

    struct st d;

    printf("Size of a, b and c: %zu\n", sizeof(d.a) + sizeof(d.b) + sizeof(d.c));
    printf("Size of struct: %zu\n", sizeof(d));
}
```

<div class="term">
<b>~]$</b> ./s6
Size of a, b and c: 13
Size of struct: 24
</div>

We can use the gcc option `-Wpadded` to get warned about padding in structures:

<div class="term">
<b>~]$</b> gcc s6.c -std=c99 -Wpadded
s6.c: In function ‘main’:
s6.c:13:17: warning: padding struct to align ‘b’ [-Wpadded]
         int64_t b;
                 ^
s6.c:15:5: warning: padding struct size to alignment boundary [-Wpadded]
     };
     ^
</div>

When warned about padding, we can see if we're able to reduce the size of the structure by rearranging items:

```c
/*
 * s7.c
 * gcc s7.c -o s7 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    struct st {
        int64_t b;
        int32_t a;
        int8_t  c;
    };

    struct st d;

    printf("Size of a, b and c: %zu\n", sizeof(d.a) + sizeof(d.b) + sizeof(d.c));
    printf("Size of struct: %zu\n", sizeof(d));
}
```

<div class="term">
<b>~]$</b> ./s7
Size of a, b and c: 13
Size of struct: 16
</div>

We reduced the size of the structure by 8 bytes by simply rearranging the items. The general rule is putting large data types first.

### In memory

Let's see how the structure is stored in memory:

```c
/*
 * s8.c
 * gcc s8.c -o s8 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    struct st {
        int64_t b;
        int32_t a;
        int8_t  c;
    };

    struct st d = {.a = 10, .b = 200, .c = 3};

    printf("(%p) d\n", (void *) &d);
    printf("(%p) d.a = %" PRId32  "\n", (void *)&d.a, d.a);
    printf("(%p) d.b = %" PRId64  "\n", (void *)&d.b, d.b);
    printf("(%p) d.c = %" PRId8   "\n", (void *)&d.c, d.c);
}
```

<div class="term">
<b>~]$</b> ./s8
(0x7ffe32d94880) d
(0x7ffe32d94888) d.a = 10
(0x7ffe32d94880) d.b = 200
(0x7ffe32d9488c) d.c = 3
</div>

Observe that the structure itself and the first element (b) have the same address, and that this is the lowest address.
The last element (c), has the highest address.

```
memory:

              high addresses
     ===========
     | stack   |
     |---------|
     |   d.c   | 0x7ffe32d9488c
     |---------|
     |   d.a   | 0x7ffe32d94888
     |---------|
     | d / d.b | 0x7ffe32d94880
     |---------|
     | |  |  | |
     | v  v  v |
     |         |
     |         |
     | ^  ^  ^ |
     | |  |  | |
     |---------|
     | heap    |
     ===========
              low addresses
```

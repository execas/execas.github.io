---
layout: post
title: Floating-point in C
date: 2019-02-28
tags: [c, programming]
---

## Floating-point intro

C defines the `float`, `double` and `long double` floating-point data types, with the following storage sizes:

```c
/*
 * a1.c
 * gcc a1.c -o a1 -std=c99
 */

#include <stdio.h>

int main()
{
    printf("sizeof(float):       %3d bits\n", (sizeof(float) * 8));
    printf("sizeof(double):      %3d bits\n", (sizeof(double) * 8));
    printf("sizeof(long double): %3d bits\n", (sizeof(long double) * 8));
}
```

<div class="term">
~]$ ./a1
sizeof(float):        32 bits
sizeof(double):       64 bits
sizeof(long double): 128 bits
</div>

> Note: Compiling the above code on a 32-bit system, or using `gcc -m32 ...` may give a long double size of 96 bits.

The maximum and minimum values a floating-point data type can store, is determined by the memory it uses for storage (as well as its representation, which will be covered later).

### Maximum and minimum values

`float.h`, part of the C Standard Library, can be used to examine some of the properties of these types, for example maximum and minimum values:

```c
/*
 * a2.c
 * gcc a2.c -o a2 -std=c99
 */

#include <stdio.h>
#include <float.h>

int main()
{
    printf("Max value\n"
           "--------------------------\n"
           "float:       %g\n"
           "double:      %lg\n"
           "long double: %Lg\n\n", FLT_MAX, DBL_MAX, LDBL_MAX);
    printf("Min value\n"
           "--------------------------\n"
           "float:       %g\n"
           "double:      %lg\n"
           "long double: %Lg\n", FLT_MIN, DBL_MIN, LDBL_MIN);
}
```

<div class="term">
~]$ ./a2
Max value
--------------------------
float:       3.40282e+38
double:      1.79769e+308
long double: 1.18973e+4932

Min value
--------------------------
float:       1.17549e-38
double:      2.22507e-308
long double: 3.3621e-4932
</div>

### Printing

In the example above we used `%g` for float, `%lg` for double and `%Lg` for the long double. These display the floating-point type in either decimal notation or scientific notation, depending on the size of the value.

For decimal notation:

- `%f` for float
- `%lf` for double
- `%Lf` for long double

For scientific notation:

- `%e` for float
- `%le` for double
- `%Le` for long double


> Note: `%[efg]` and `%l[efg]` mean the same thing to the printf-function, the "l" is ignored. A float argument is promoted to a double in printf, and `%[efg]` can correctly display a double value. But make a habit out of using `%l[efg]` for double anyway, since it doesn't hurt, makes the code more explicit, and is more in line with the specifiers `scanf` requires.

> **Warning:** In C89, `%l[efg]` may cause undefined behavior.

### Precision

When a real number is stored in a floating-point data type, precision may be lost. More bits gives the possibility for greater precision, and this can be illustrated by comparing a float and a double both set to 1.1:

```c
/*
 * a3.c
 * gcc a3.c -o a3 -std=c99
 */

#include <stdio.h>

int main()
{
    float f = 1.1;
    double d = 1.1;
    printf("f %s= d\n", (f == d) ? "=" : "!");
}
```

```bash
~]$ ./a3
f != d
```

The comparison fails since the double does a better job at representing 1.1:

```c
/*
 * a4.c
 * gcc a4.c -o a4 -std=c99
 */

#include <stdio.h>

int main()
{
    float f = 1.1f;
    double d = 1.1;
    printf("f = %.20f\n", f);
    printf("d = %.20lf\n", d);
}
```

```bash
~]$ ./a4
f = 1.10000002384185791016
d = 1.10000000000000008882
```

The two numbers above are clearly not equal. They are also clearly not exactly 1.1.

> Note: Be careful when comparing numbers stored in different data types. Never compare values stored in different floating-point data types.

### Suffixing floating constants

In the examples above, the floating constant (also called *floating literal*) assigned to `float f` has an "f" suffixed. Why?

An unsuffixed floating constant has the default type double. The "f" suffix gives it type float, the "L" suffix gives it type long double.

> Note: we can use "F" and "l" instead, but "f" and "L" are easier to read and more in tune with the printf format specifier syntax.

We should make a habit out of suffixing values assigned to a float with "f" and values assigned to a long double with "L", or risk conversion errors:

```c
/*
 * a5.c
 * gcc a5.c -o a5 -std=c99
 */

#include <stdio.h>

int main()
{
    double d = 1.1;
    long double l = 1.1;
    long double ls = 1.1L;
    printf("d:  %.25lf\n", d);
    printf("l:  %.25Lf\n", l);
    printf("ls: %.25Lf\n", ls);
}
```

```bash
~]$ ./a5
d:  1.1000000000000000888178420
l:  1.1000000000000000888178420
ls: 1.1000000000000000000216840
```

Notice that *l* is equal to *d*, since the unsuffixed floating constant was first converted to a double, while *ls*, a true long double, is the better representation of 1.1.

### Casting

Casting (type casting) allows us convert a variable temporarily. This can be useful or necessary when doing for example assignments, calculations or printing of data.

If we for cast a float to a 32 bit integer, we are left with the integer part:

```c
/*
 * a6.c
 * gcc a6.c -o a6 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    float f1 = 100.13f;
    float f2 = 100.77f;
    printf("%" PRId32 "\n", (int32_t)f1);
    printf("%" PRId32 "\n", (int32_t)f2);
}
```

```bash
~]$ ./a6
100
100
```

> Sidenote: If we excluded the casting to `int32_t` above, we would get undefined behavior since the format specifier would not match the data type of the passed variable. Always make sure that the conversion character matches the data type of the argument.

We can lose data during casting, but we can never gain (or get back):

```c
/*
 * a7.c
 * gcc a7.c -o a7 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    long double ld = 123.456789L;
    float fl = (float)ld;
    printf("long double:\t\t%.12Lf\n", ld);
    printf("float from ld:\t\t%.12lf\n", fl);
    printf("long double from fl:\t%.12f\n", (long double)fl);
}
```

```
~]$ ./a7
long double:            123.456789000000
float from ld:          123.456787109375
long double from fl:    123.456787109375
```

## Single-precision floating-point

The float data type is in the IEEE754 single-precision binary floating-point format (32 bits) on most systems.

Let's store a value in a `float`, then examine it as a 32-bit signed integer:

```c
/*
 * f1.c
 * gcc f1.c -o f1 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    float data = 100.13;
    printf("%" PRId32 "\n", *((int32_t *)&data));
}
```

```bash
~]$ ./f1
1120420495
```

Now, let's have a look at this data in binary:

```c
/*
 * f2.c
 * gcc f2.c -o f2 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    int32_t v = 1120420495;
    for (int i = 31; i >= 0; i--)
        printf("%d", (v & (1 << i)) != 0);
}
```

```
$ ./f2
01000010110010000100001010001111
```

> Note: The code above does bit-wise comparison to display the binary value of a 32 bit integer. It contains a subtle potential bug that we will investigate when we look at doubles.

### How a float is represented

The above output has the following format:

- The first bit is for the sign. 0 means positive.
- The next eight bits is for the exponent.
- The last twenty-three bits is for the mantissa.

The floating-point calculation is:

(-1)^*sign-bit* * 1.*mantissa* * 2^(*exponent* - *bias*)

### Calculating the exponent

- The 8 bits reserved for the exponent can represent 0-255.
- 0-127 represent negative exponents.
- 128-255 represent positive exponents.
- The exponent is calculated by converting to decimal, then subtracting the *bias*.
- The bias is 127 (2⁸/2 - 1)

Converting the eight bits to decimal:
10000101 = 128 + 4 + 1 = 133

133 represents the positive exponent 6 (since 133 - 127 = 6).

> Note: By adding a bias to the exponent, both negative and positive values can be stored in memory without using a bit to represent sign -- just subtract the bias afterwards.

> Note: 127 represents the exponent 0, 0 represents the exponent -127

### Calculating the mantissa

Before calculating the mantissa, there is an omitted "1." we need to add back (as seen in the floating-point calculation above):

```
1.10010000100001010001111
```

Now, from left to right (including the bit before the decimal point),the bits of this n-bit string represent the numbers [1, 1/2¹,1/2²,..., 1/2ⁿ]. To calculate the mantissa, the fractions corresponding to a 1 are summed.

### Code to illustate convertion from binary to single-precision floating-point

```c
/*
 * f3.c
 * gcc f3.c -o f3 -std=c99 -lm
 */

#include <stdio.h>
#include <inttypes.h>
#include <math.h>

int main()
{
    float v = 100.13f;

    /* convert to binary */
    uint8_t width = sizeof(v) * 8;
    uint8_t b[width];

    for (int i = 0; i < width; i++)
        b[i] = (*(int32_t *)&v & (1 << (width - 1 - i))) != 0;

    /* sign */
    uint8_t s = b[0];

    /* exponent */
    int8_t e = -127;
    uint8_t ew = 8; /* exponent width */

    for (int i = 1; i <= ew; i++)
        e += pow(2, (ew - i)) * b[i];
    printf("Exponent: %" PRId16 "\n\n", e);

    /* mantissa */
    float m = 0.0f;
    float frac = 0.0f;
    b[ew] = 1; /* add the missing 1. */

    printf("Mantissa built with:\n");
    for (int i = ew; i < width; i++) {
        frac = 1.0 / pow(2, i - ew);
        m += frac * b[i];
        if (b[i])
            printf("%.8f\n", frac);
    }
    printf("Mantissa: %.8f", m);

    /* result */
    float result = pow(-1, s) * m * pow(2, e);
    printf("\n\nResult: (-1)^%" PRIu8 " * %.8f * 2^%" PRId8 " = %.8f\n", \
            s, e, m, result);
}
```

```bash
~] ./f3
Exponent: 6

Mantissa built with:
1.00000000
0.50000000
0.06250000
0.00195312
0.00006104
0.00001526
0.00000095
0.00000048
0.00000024
0.00000012
Mantissa: 1.56453121

Result: (-1)^0 * 1.56453121 * 2^6 = 100.12999725
```

### Precision

We started with 100.13, but ended up with 100.12999725. Did we loose some precision during our calculations?
No, and the code below illustrates that this precison was lost at the moment we set the float's value.

```c
/*
 * f4.c
 * gcc f4.c -o f4 -std=c99
 */

#include <stdio.h>

int main()
{
    float a = 100.13;
    printf("a = %.8f\n", a);
}
```

```bash
~]$ ./f4
a = 100.12999725
```
> Note: Repeated calculations using floating-point numbers can cause errors to accumulate.

## Double-presicion floating-point

The double data type is in the IEEE754 double-precision binary floating-point format (64 bits) on most systems.

Let's store a value in a `double`, then examine it as a 64-bit signed integer:

```c
/*
 * d1.c
 * gcc d1.c -o d1 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    double data = -100.13;
    printf("%" PRId64 "\n", *((int64_t *)&data));
}
```

```bash
~]$ ./d1
-4586625597563396424
```

### Binary (and some undefined behavior)

Now, let's examine this data in binary, using the same bit-shifting method we used when looking at floats:

```c
/*
 * d2.c
 * gcc d2.c -o d2 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    int64_t v = -4586625597563396424;
    for (int i = 63; i >= 0; i--)
        printf("%d", (v & (1 << i)) != 0);
}
```

```bash
~]$ ./d2
1110101110000101000111101011100011101011100001010001111010111000
```

Something went wrong, even though it may not be obvious to us at first glance, or even without first trying to convert it to a double. 

Look at the output, the same 32-bit binary string is repeated twice:

```
11101011100001010001111010111000 \
11101011100001010001111010111000
```

What happened here?

It's the integer literal 1's fault. The default integer literal is 32 bits, and we shift bits by more than this for the first half of the for-loop.

> **Warning:** Shifting bits with values greater than or equal to the width of the data type operated on causes undefined behavior.

"Undefined" does not necessarily mean "unexplainable". In some cases - such as ours - it may be both obvious and logical (but certainly not acceptable or safe for use in production code anyhow).

On the system and compiler used, shifting by 63-32 is equal to shifting by 31-0, and that's why the 32-bit binary string repeats twice:

```c
/*
 * d3.c
 * gcc d3.c -o d3 -std=c99
  */

#include <stdio.h>

int main()
{
    for (int i = 31; i >= 0; i--) {
        int j = 63 - (31 - i);
        printf("%d %11d | %2d %11d\n", j, (1 << j), i, (1 << i));
    }
}
```

```
~]$ ./d3
63 -2147483648 | 31 -2147483648
62  1073741824 | 30  1073741824
61   536870912 | 29   536870912
60   268435456 | 28   268435456
59   134217728 | 27   134217728
...
37          32 |  5          32
36          16 |  4          16
35           8 |  3           8
34           4 |  2           4
33           2 |  1           2
32           1 |  0           1
```

But we should never rely on this! When we shift by values greater than the width of the data type in the example below, we get 0 as a result:

```c
/*
 * d4.c
 * gcc d4.c -o d4 -std=c99
 */

#include <stdio.h>

int main()
{
    printf("%d\n", (1 << 63));
    printf("%d\n", (1 << 32));
    printf("%d\n", (1 << 31));
    printf("%d\n", (1 << 0));
}
```

```
~]$ ./d4
0
0
-2147483648
1
```

Back to the binary conversion. How can we correct it?

We make sure the 1 we shift is cast to a 64-bit integer, suffixed by an "L" or accessed through a 64-bit integer variable to prevent it from being the default 32-bit integer.

```c
/*
 * d5.c
 * gcc d5.c -o d5 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    int64_t v = -4586625597563396424;
    for (int i = 63; i >= 0; i--)
        printf("%" PRId8, (v & (1L << i)) != 0);
}
```

```
~]$ ./d5
1100000001011001000010000101000111101011100001010001111010111000
```

### How a double is represented

The above output has the following format:

- The first bit is for the sign. 1 means negative.
- The next eleven bits is for the exponent.
- The last fifty-two bits is for the mantissa.

The calculation to convert from binary to double is:

(-1)^*sign-bit* * 1.*mantissa* * 2^(*exponent* - *bias*)

The calculation is identical to the one used for floats, and so is the explanation below on how to convert from binary to double, except for the number of bits and what they represent.

### Calculating the exponent

- The 11 bits reserved for the exponent can represent 0-2047.
- 0-1023 represent negative exponents.
- 1024-2047 represent positive exponents.
- The exponent is calculated by converting to decimal, then subtracting the *bias*.
- The bias is 1023 ((2^11 / 2) - 1)

Converting the eleven bits to decimal:
10000000101 = 1024 + 4 + 1 = 1029

1029 represents the positive exponent 6 (since 1029-1023 = 6).

> Note: 1023 represents the exponent 0, 0 represents the exponent -1023

### Calculating the mantissa

Before calculating the mantissa, there is an omitted "1." we need to add back (as seen in the floating-point calculation above):

```
1.1001000010000101000111101011100001010001111010111000
```

Now, from left to right (including the bit before the decimal point),the bits of this n-bit string represent the numbers [1, 1/2¹,1/2²,..., 1/2ⁿ]. To calculate the mantissa, the fractions corresponding to a 1 are summed.

### Code to illustate convertion from binary to double-precision floating-point

```c
/*
 * d6.c
 * gcc d6.c -o d6 -std=c99 -lm
 */

#include <stdio.h>
#include <inttypes.h>
#include <math.h>

int main()
{
    double v = -100.13;

    /* convert to binary */
    uint8_t width = sizeof(v) * 8;
    uint8_t b[width];

    for (int i = 0; i < width; i++)
        b[i] = (*(int64_t *)&v & (1l << (width - 1 - i))) != 0;

    /* sign */
    uint8_t s = b[0];

    /* exponent */
    int16_t e = -1023;
    uint8_t ew = 11; /* exponent width */

    for (int i = 1; i <= ew; i++)
        e += pow(2, (ew - i)) * b[i];
    printf("Exponent: %" PRId16 "\n\n", e);

    /* mantissa */
    double m = 0;
    double frac = 0;
    b[ew] = 1; /* add the missing 1. */

    printf("Mantissa built with:\n");
    for (int i = ew; i < width; i++) {
        frac = 1.0 / pow(2, i - ew);
        m += frac * b[i];
        if (b[i])
            printf("%.15lf\n", frac);
    }
    printf("Mantissa: %.8lf", m);

    /* result */
    double result = pow(-1, s) * m * pow(2, e);
    printf("\n\nResult: (-1)^%" PRIu8 " * %.8lf * 2^%" PRId16 " = %.15lf\n", \
            s, e, m, result);
}
```

```bash
~]$ ./d6
Exponent: 6

Mantissa built with:
1.000000000000000
0.500000000000000
0.062500000000000
0.001953125000000
0.000061035156250
0.000015258789062
0.000000953674316
0.000000476837158
0.000000238418579
0.000000119209290
0.000000029802322
0.000000007450581
0.000000003725290
0.000000001862645
0.000000000058208
0.000000000014552
0.000000000000909
0.000000000000455
0.000000000000227
0.000000000000114
0.000000000000028
0.000000000000007
0.000000000000004
0.000000000000002
Mantissa: 1.56453125

Result: (-1)^1 * 1.56453125 * 2^6 = -100.129999999999995
```
Even more precision is offered by the long double data type.

## The long double

The long double is only required by the C language to be at least as precise as the double. With gcc on x86 architecture `long double` is the 80-bit extended precision type. The storage is typically 96 or 128 bits, due to padding which maintains data structure alignment. This means that a lot of storage space is essentially wasted.

> Note: The `long double` may be in the IEEE 754 quadruple-precision format (128 bits), but only with a few systems and compilers, since hardware support for quadruple-precision is not very common.

> Note: gcc has a quadruple-precicion type called `__float128`.

Let's store a value in a `long double`, then examine each of the 12 or 16 bytes:


```c
/*
 * l1.c
 * gcc l1.c -o l1 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    long double v = 100.13L;
    for (size_t i = 0; i < sizeof(v); i++) {
        uint8_t b = *(((uint8_t *)&v)+i);
        printf("%" PRId8 " ", b);
    }
}
```

```bash
~]$ ./l1
143 194 245 40 92 143 66 200 5 64 0 0 0 0 0 0
```

The size of the long double is 16 bytes (128 bits) on this system. The long double can store 80 bits, which means 48 bits (6 bytes) are padded. As we can see from the output above, this must be the last 6 bytes (which are actually the first 6 bytes since the above output is Little-endian).

Let's examine these bytes, excluding the last 6, in binary:

```c
/*
 * l2.c
 * gcc l2.c -o l2 -std=c99
 */

#include <stdio.h>
#include <inttypes.h>

int main()
{
    uint8_t b[] = {143,194,245,40,92,143,66,200,5,64};
    for (size_t i = 0; i < sizeof(b); i++) {
        for (int j = 7; j >= 0; j--)
            printf("%d", (b[i] & (1 << j)) != 0);
        printf(" ");
    }
}
```

``` bash
~]$ ./l2
10001111 11000010 11110101 00101000 01011100 \
10001111 01000010 11001000 00000101 01000000
```

The output is in Little-endian order, so let's reverse the byte order

```
01000000 00000101 11001000 01000010 10001111 \
01011100 00101000 11110101 11000010 10001111
```

- The first bit is for the sign. 0 means positive.
- The next fifteen bits is for the exponent.
- The next bit is the integer part.
- The last sixty-three bits is for the mantissa.

The floating-point calculation is:

(-1)^*sign-bit* * *integer-bit*.*mantissa* * 2^(*exponent* - *bias*)

The calculation is identical to the one used for floats and doubles, and so is the explanation below on how to convert from binary to long double, except for the number of bits and what they represent, and the fact that there is no "hidden bit" (explained soon).

### Calculating the exponent

- The 15 bits reserved for the exponent can represent 0-32767.
- 0-16383 represent negative exponents.
- 16384-32767 represent positive exponents.
- The exponent is calculated by converting to decimal, then subtracting the *bias*.
- The bias is 16383 ((2^15 / 2) - 1)

Converting the fifteen bits to decimal:
100000000000101 = 16384 + 4 + 1 = 16389

16389 represents the positive exponent 6 (since 16389 - 16383 = 6).

### Calculating the mantissa

For floats and doubles, there is an omitted "1" that we have to add back before continuing our calculation, but for long doubles there is no "hidden bit". This bit is in the data, between the exponent and fraction bits:

```
1.100100001000010100011110101110000101000111101011100001010001111
```

Now, from left to right (including the bit before the decimal point),the bits of this n-bit string represent the numbers [1, 1/2¹,1/2²,..., 1/2ⁿ]. To calculate the mantissa, the fractions corresponding to a 1 are summed.

### Code to illustate convertion from binary to long double

```c
/*
 * l3.c
 * gcc l3.c -o l3 -std=c99 -lm
 */

#include <stdio.h>
#include <inttypes.h>
#include <math.h>

int main()
{
    long double v = -100.13L;

    /* convert to binary */
    uint8_t b[80];
    uint8_t *vp = (uint8_t *)&v;
    vp += 9; /* 10 first bytes in Little-endian are data, rest is padding */

    for (int i = 0; i < 10; i++) {
        for (int j = 0; j < 8; j++)
            b[j + (i*8)] = (*(vp-i) & (1 << (7-j))) !=0;
    }

    /* sign */
    uint8_t s = b[0];

    /* exponent */
    int16_t e = -16383;
    uint8_t ew = 15; /* exponent width */

    for (int i = 1; i <= ew; i++)
        e += pow(2, (ew - i)) * b[i];
    printf("Exponent: %" PRId16 "\n\n", e);

    /* mantissa */
    long double m = 0.0L;
    long double frac = 0.0L;
    uint8_t ip = ew+1; /* integer part position */

    printf("Mantissa built with:\n");
    for (int i = ip; i < 80; i++) {
        frac = 1.0L / pow(2, i - ip);
        m += frac * b[i];
        if (b[i])
            printf("%.28Lf\n", frac);
    }
    printf("Mantissa: %.8Lf", m);

    /* result */
    long double result = pow(-1, s) * m * powl(2, e);
    printf("\n\nResult: (-1)^%" PRIu8 " * %.8Lf * 2^%" PRId16 " = %.28Lf\n", \
            s, e, m, result);
}
```

```
Exponent: 6

Mantissa built with:
1.0000000000000000000000000000
0.5000000000000000000000000000
0.0625000000000000000000000000
0.0019531250000000000000000000
0.0000610351562500000000000000
0.0000152587890625000000000000
0.0000009536743164062500000000
0.0000004768371582031250000000
0.0000002384185791015625000000
0.0000001192092895507812500000
0.0000000298023223876953125000
0.0000000074505805969238281250
0.0000000037252902984619140625
0.0000000018626451492309570312
0.0000000000582076609134674072
0.0000000000145519152283668518
0.0000000000009094947017729282
0.0000000000004547473508864641
0.0000000000002273736754432321
0.0000000000001136868377216160
0.0000000000000284217094304040
0.0000000000000071054273576010
0.0000000000000035527136788005
0.0000000000000017763568394003
0.0000000000000000555111512313
0.0000000000000000138777878078
0.0000000000000000008673617380
0.0000000000000000004336808690
0.0000000000000000002168404345
0.0000000000000000001084202172
Mantissa: 1.56453125

Result: (-1)^1 * 1.56453125 * 2^6 = -100.1299999999999999975019981946
```

### float, double or long double?

We choose a data type based on our needs. What precision is actually needed? Greater precison comes at the cost of more storage space and slower operations.

> Note: `double` is the default floating-point type in a lot of programming languages, and is often thought of as the go-to type for floating-point.

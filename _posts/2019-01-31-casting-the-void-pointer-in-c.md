---
layout: post
title: Casting the void pointer in c
date: 2019-01-31
tags: [c, programming]
---

This article shows the flexibility of memory on the heap and the `void` pointer.

In the examples below, 100 bytes of data is allocated on the heap using `malloc` with the returned pointer is stored in `void *data`.

### Example 1

The first example shows how we can use dereferencing to write and read an unsigned 4 byte int value to memory:

```c
/*
 * e1.c
 * gcc e1.c -o e1 -std=c11
 */

#include <stdio.h>
#include <stdlib.h>
#include <inttypes.h>

int main()
{
    /* allocate 100 bytes of memory */
    void *data = malloc(100);
    *((uint32_t *)data) = 3333;
    printf("%" PRIu32 "\n", *((uint32_t *)data));
    free(data);
}
```

 ```bash
 $ ./e1
 3333
 ```

The first byte (byte 0) of the heap memory is pointed to by `*void data`. We use `*(uint32_t *)` to cast and dereference the pointer. Casting makes sure we read/write the correct amount of bytes, and dereferencing let's us access the value stored at the pointer.

### Example 2

Casting is very powerful. In the next example we do the same as in the first, but we only read back a single byte when printing:


```c
/*
 * e2.c
 * gcc e2.c -o e2 -std=c11
 */

#include <stdio.h>
#include <stdlib.h>
#include <inttypes.h>

int main()
{
    /* allocate 100 bytes of memory */
    void *data = malloc(100);
    *((uint32_t *)data) = 3333;
    printf("%" PRIu8 "\n", *((uint8_t *)data));
    free(data);
}
```

```bash
$ ./e2
5
```

The output is 5, why?

1. 3333 is 00000000 00000000 00001101 00000101 when expressed as 32 bits of binary.
2. in Little-endian this becomes 00000101 00001101 00000000 00000000
3. the first byte is 00000101
4. 00000101 is 5 in decimal

### Example 3

Let's confirm the above by printing the second byte (00001101), which should be 13 in decimal:

```c
/*
 * e3.c
 * gcc e3.c -o e3 -std=c11
 */

#include <stdio.h>
#include <stdlib.h>
#include <inttypes.h>

int main()
{
    /* allocate 100 bytes of memory */
    void *data = malloc(100);
    *((uint32_t *)data) = 3333;
    printf("%" PRIu8 "\n", *(((uint8_t *)data)+1));
    free(data);
}
```

```bash
$ ./e3
13
```

The only difference in the code above is found in the `printf` function, where we access the value at the second byte by using casting with a `+1` offset. An alternative, and maybe preferable, syntax would be using `((uint8_t *)data)[1]`.

### Example 4

Lets try setting 4 bytes, then reading them as a `uint32_t`:

```c
/*
 * e4.c
 * gcc e4.c -o e4 -std=c11
 */

#include <stdio.h>
#include <stdlib.h>
#include <inttypes.h>

int main()
{
    /* allocate 100 bytes of memory */
    void *data = malloc(100);
    *((uint8_t *)data) = 255;
    *(((uint8_t *)data)+1) = 255;
    *(((uint8_t *)data)+2) = 255;
    *(((uint8_t *)data)+3) = 0;
    printf("%" PRIu32 "\n", *((uint32_t *)data));
    free(data);
}
```

```bash
$ ./e4
16777215
```

In Little-endian, the bytes we set are represented as eight 0 bits followed by twenty-four 1 bits.
24 bits can represent unsigned numbers 0-16777215, and since all of them are set, we of course get the maximum value.

### Example 5

In the example below, we set several bytes with decimal values, then print them as a string:


```c
/*
 * e5.c
 * gcc e5.c -o e5 -std=c11
 */

#include <stdio.h>
#include <stdlib.h>
#include <inttypes.h>

int main()
{
    /* allocate 100 bytes of memory */
    void *data = malloc(100);
    *((uint8_t *)data) = 72;
    *(((uint8_t *)data)+1) = 101;
    *(((uint8_t *)data)+2) = 108;
    *(((uint8_t *)data)+3) = 108;
    *(((uint8_t *)data)+4) = 111;
    *(((uint8_t *)data)+5) = 0;
    printf("%s\n", (uint8_t *)data);
    free(data);
}
```

```bash
$ ./e5
Hello
```

The output is a result of setting the bytes to the decimal value of the wanted ASCII characters (with the last byte set to 0 to mark the end of the string), and using the `"%s` format specifer in `printf()`. Note that the void pointer is cast `(uint8_t *)`, since the format specifier expects a `char *`, not an `int` (which we would get if we dereferenced the pointer).

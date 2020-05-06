---
layout: post
title: "Buffer overflow - countermeasure 2: ASLR"
date: 2020-01-15
tags: [security, linux, programming, vulnerabilities]
---

Address space layout randomization (ASLR) randomizes the position of segments in the process's address space. This makes it difficult to exploit a vulnerable program by for example jumping to functions, chaining instructions togheter (ROP) and executing injected code, since all of these methods require the addresses of functions, instructions, environment variables, the stack and so on.

ASLR is enabled by default on Linux. We have disabled ASLR in all examples so far by using:

```bash
$ sudo sh -c "echo 0 > /proc/sys/kernel/randomize_va_space"
```

It can be re-enabled by rebooting or by using:

```bash
$ sudo sh -c "echo 2 > /proc/sys/kernel/randomize_va_space"
```

> We can also use `sysctl -a -r randomize` to view its value, and `sudo sysctl kernel.randomize_va_space=1` (use tab completion) to activate.

> ASLR on Linux also has a 1 value in addition to 0 and 2. It enables ASLR, but does not randomize the address space for data segments.

### What is randomized?

Notice that when ASLR is disabled, shared objects have the same address each time:

```
$ ldd bo_vuln1
    linux-vdso.so.1 (0x00007ffff7fd1000)
    libc.so.6 => /lib64/libc.so.6 (0x00007ffff7de9000)
    /lib64/ld-linux-x86-64.so.2 (0x00007ffff7fd2000)
$ ldd bo_vuln1
    linux-vdso.so.1 (0x00007ffff7fd1000)
    libc.so.6 => /lib64/libc.so.6 (0x00007ffff7de9000)
    /lib64/ld-linux-x86-64.so.2 (0x00007ffff7fd2000)
```

When ASLR is enabled, the addresses changes each time:

```
$ ldd bo_vuln1
    linux-vdso.so.1 (0x00007fff609f7000)
    libc.so.6 => /lib64/libc.so.6 (0x00007f0329045000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f032922a000)
$ ldd bo_vuln1
    linux-vdso.so.1 (0x00007fffb7945000)
    libc.so.6 => /lib64/libc.so.6 (0x00007fa68c9e3000)
    /lib64/ld-linux-x86-64.so.2 (0x00007fa68cbc8000)
```

An executable itself will not be at a random address unless compiled as a *position-independent executable* (PIE).

```c
/*
 * mainaddr.c
 * gcc mainaddr.c
 */
#include <stdio.h>

int main() {
    printf("%p\n", main);
}
```

> The above code prints the address of `main`.

When compiled "normally" using `gcc` with no options:

```bash
$ ./a.out; ./a.out
0x401126
0x401126
```

When compiled using `gcc mainaddr.c -fpic -pie`:

```bash
$ ./a.out; ./a.out
0x55736618d139
0x55e043928139
```

> In gdb, `disable-randomization` is on by default, so we need to turn it off to see the effects of an active ASLR on the pie executable.

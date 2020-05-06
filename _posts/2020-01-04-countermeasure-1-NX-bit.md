---
layout: post
title: "Buffer overflow - countermeasure 1: NX bit"
date: 2020-01-04
tags: [security, linux, programming, vulnerabilities]
---

The NX bit (no-execute) is a countermeasure that prevents execution of code on the stack. If a program's stack is *not* marked as executable, the exploits we've looked at so far will not work. On modern systems and processors, a non-executable stack is the default.

> No-execute has many names, including *data execution prevention* (DEP), *W^X* and *executable-space protection*.

In the previous articles we compiled all our programs with `-zexecstack`. When gcc is given a `-z` followed by a keyword, both are passed directly to the linker. The `execstack` keyword "marks the object as requiring executable stack" (from `ld` manpage).

If we take the `bo_vuln1` program from the previous section, and compile it again without the linker keyword, the exploit will stop working:

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

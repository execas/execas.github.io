---
layout: post
title: The ELF file format 5. Basic ELF tools and commands
date: 2019-04-21
tags: [c, linux, programming]
---

This post will cover some of the basic tools and commands we can use to investigate ELF files.

## Displaying information about ELF files

### readelf

We have already covered `readelf -S` and `readelf -l`, which diplay information about the section headers and program/segment headers, respectively. These are often used with the `-W` option to produce more readable output on modern terminals. We also saw how `readelf -h` could be used to display the information in the ELF header.

### objdump

The `objdump` tool can display a lot of the same information as `readelf`:

Sections:

```bash
$ objdump -hw c1

c1:     file format elf64-x86-64

Sections:
Idx Name               Size      VMA               LMA               File off  Algn  Flags
0 .interp            0000001c  0000000000400238  0000000000400238  00000238  2**0  CONTENTS, ALLOC, LOAD, READONLY, DATA
1 .note.ABI-tag      00000020  0000000000400254  0000000000400254  00000254  2**2  CONTENTS, ALLOC, LOAD, READONLY, DATA
2 .note.gnu.build-id 00000024  0000000000400274  0000000000400274  00000274  2**2  CONTENTS, ALLOC, LOAD, READONLY, DATA
3 .gnu.hash          0000001c  0000000000400298  0000000000400298  00000298  2**3  CONTENTS, ALLOC, LOAD, READONLY, DATA
4 .dynsym            00000060  00000000004002b8  00000000004002b8  000002b8  2**3  CONTENTS, ALLOC, LOAD, READONLY, DATA
5 .dynstr            0000003f  0000000000400318  0000000000400318  00000318  2**0  CONTENTS, ALLOC, LOAD, READONLY, DATA
6 .gnu.version       00000008  0000000000400358  0000000000400358  00000358  2**1  CONTENTS, ALLOC, LOAD, READONLY, DATA
7 .gnu.version_r     00000020  0000000000400360  0000000000400360  00000360  2**3  CONTENTS, ALLOC, LOAD, READONLY, DATA
8 .rela.dyn          00000018  0000000000400380  0000000000400380  00000380  2**3  CONTENTS, ALLOC, LOAD, READONLY, DATA
9 .rela.plt          00000030  0000000000400398  0000000000400398  00000398  2**3  CONTENTS, ALLOC, LOAD, READONLY, DATA
10 .init              0000001a  00000000004003c8  00000000004003c8  000003c8  2**2  CONTENTS, ALLOC, LOAD, READONLY, CODE
11 .plt               00000030  00000000004003f0  00000000004003f0  000003f0  2**4  CONTENTS, ALLOC, LOAD, READONLY, CODE
12 .plt.got           00000008  0000000000400420  0000000000400420  00000420  2**3  CONTENTS, ALLOC, LOAD, READONLY, CODE
13 .text              000001a2  0000000000400430  0000000000400430  00000430  2**4  CONTENTS, ALLOC, LOAD, READONLY, CODE
14 .fini              00000009  00000000004005d4  00000000004005d4  000005d4  2**2  CONTENTS, ALLOC, LOAD, READONLY, CODE
15 .rodata            00000014  00000000004005e0  00000000004005e0  000005e0  2**3  CONTENTS, ALLOC, LOAD, READONLY, DATA
16 .eh_frame_hdr      00000034  00000000004005f4  00000000004005f4  000005f4  2**2  CONTENTS, ALLOC, LOAD, READONLY, DATA
17 .eh_frame          000000ec  0000000000400628  0000000000400628  00000628  2**3  CONTENTS, ALLOC, LOAD, READONLY, DATA
18 .init_array        00000008  0000000000600e10  0000000000600e10  00000e10  2**3  CONTENTS, ALLOC, LOAD, DATA
19 .fini_array        00000008  0000000000600e18  0000000000600e18  00000e18  2**3  CONTENTS, ALLOC, LOAD, DATA
20 .jcr               00000008  0000000000600e20  0000000000600e20  00000e20  2**3  CONTENTS, ALLOC, LOAD, DATA
21 .dynamic           000001d0  0000000000600e28  0000000000600e28  00000e28  2**3  CONTENTS, ALLOC, LOAD, DATA
22 .got               00000008  0000000000600ff8  0000000000600ff8  00000ff8  2**3  CONTENTS, ALLOC, LOAD, DATA
23 .got.plt           00000028  0000000000601000  0000000000601000  00001000  2**3  CONTENTS, ALLOC, LOAD, DATA
24 .data              00000004  0000000000601028  0000000000601028  00001028  2**0  CONTENTS, ALLOC, LOAD, DATA
25 .bss               00000004  000000000060102c  000000000060102c  0000102c  2**0  ALLOC
26 .comment           0000002d  0000000000000000  0000000000000000  0000102c  2**0  CONTENTS, READONLY
```

> Note that `objdump` is missing som information, e.g. `.shstrtab` and `.symtab`, which are visible in the `readelf` output. `readelf` should be preferred over `objdump` since the latter may miss or misrepresent information due to its reliance on `libbfd`.

Program headers:

```bash
$ objdump -p c1

c1:     file format elf64-x86-64

Program Header:
PHDR off    0x0000000000000040 vaddr 0x0000000000400040 paddr 0x0000000000400040 align 2**3
filesz 0x00000000000001f8 memsz 0x00000000000001f8 flags r-x
INTERP off    0x0000000000000238 vaddr 0x0000000000400238 paddr 0x0000000000400238 align 2**0
filesz 0x000000000000001c memsz 0x000000000000001c flags r--
LOAD off    0x0000000000000000 vaddr 0x0000000000400000 paddr 0x0000000000400000 align 2**21
filesz 0x0000000000000714 memsz 0x0000000000000714 flags r-x
LOAD off    0x0000000000000e10 vaddr 0x0000000000600e10 paddr 0x0000000000600e10 align 2**21
filesz 0x000000000000021c memsz 0x0000000000000220 flags rw-
DYNAMIC off    0x0000000000000e28 vaddr 0x0000000000600e28 paddr 0x0000000000600e28 align 2**3
filesz 0x00000000000001d0 memsz 0x00000000000001d0 flags rw-
NOTE off    0x0000000000000254 vaddr 0x0000000000400254 paddr 0x0000000000400254 align 2**2
filesz 0x0000000000000044 memsz 0x0000000000000044 flags r--
EH_FRAME off    0x00000000000005f4 vaddr 0x00000000004005f4 paddr 0x00000000004005f4 align 2**2
filesz 0x0000000000000034 memsz 0x0000000000000034 flags r--
STACK off    0x0000000000000000 vaddr 0x0000000000000000 paddr 0x0000000000000000 align 2**4
filesz 0x0000000000000000 memsz 0x0000000000000000 flags rw-
RELRO off    0x0000000000000e10 vaddr 0x0000000000600e10 paddr 0x0000000000600e10 align 2**0
filesz 0x00000000000001f0 memsz 0x00000000000001f0 flags r--

...
```

Some header info is available using the `objdump -f <file>` command.

### file

The `file` command is a great tool to determine file type, and is often one of the first tools run against unknown files. When used on and ELF file, it displays a lot of interesting information.

```bash
$ file c1
c1: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for
GNU/Linux 2.6.32, BuildID[sha1]=d4a02535872e7dbe09daed8ab4b20128d7457f62, not stripped
```

## Displaying the contents of ELF files

### Dumping hex

We can use `readelf -x <section name/nr> <file>` to hexdump a section:

```
$ readelf -x .text c1

Hex dump of section '.text':
0x00400430 31ed4989 d15e4889 e24883e4 f0505449 1.I..^H..H...PTI
0x00400440 c7c0d005 400048c7 c1600540 0048c7c7 ....@.H..`.@.H..
0x00400450 1d054000 e8b7ffff fff4660f 1f440000 ..@.......f..D..
0x00400460 b8371060 0055482d 30106000 4883f80e .7.`.UH-0.`.H...
...
```

An alternative is `objdump -s -j <section name> <file>`:

```bash
$ objdump -s -j .text c1

c1:     file format elf64-x86-64

Contents of section .text:
400430 31ed4989 d15e4889 e24883e4 f0505449  1.I..^H..H...PTI
400440 c7c0d005 400048c7 c1600540 0048c7c7  ....@.H..`.@.H..
400450 1d054000 e8b7ffff fff4660f 1f440000  ..@.......f..D..
400460 b8371060 0055482d 30106000 4883f80e  .7.`.UH-0.`.H...
...
```

The `hexdump`, `xxd` and `od` commands can also be used.

## Dumping strings from ELF files

### strings

The `strings` command can print strings of printable characters from any file.

```bash
$ strings c1
/lib64/ld-linux-x86-64.so.2
libc.so.6
printf
__libc_start_main
__gmon_start__
GLIBC_2.2.5
UH-0
...
```

The default is at least 4 printable characters for a byte sequence to be considered a string.

The option `-a` tells `strings` to scan the whole file (not just those in loadable, initialized data sections). See `man strings`.

### readelf

We can use `readelf -p <section name/number> <file>` to display printable string of a section along with their offset in the section.

```bash
$ readelf -p .shstrtab c1
String dump of section '.shstrtab':
[     1]  .symtab
[     9]  .strtab
[    11]  .shstrtab
[    1b]  .interp
[    23]  .note.ABI-tag
...
```

Let's do the following to deepen our understanding of this:

1) Find section header string table header:

```bash
$ readelf -h c1
...
Start of section headers:          6456 (bytes into file)
...
Size of section headers:           64 (bytes)
...
Section header string table index: 30
```

2) View section header string table header:

```bash
$ xxd -s `python -c "print 6456 + (30*64)"` c1
000020b8: 1100 0000 0300 0000 0000 0000 0000 0000  ................
000020c8: 0000 0000 0000 0000 2918 0000 0000 0000  ........).......
000020d8: 0c01 0000 0000 0000 0000 0000 0000 0000  ................
000020e8: 0100 0000 0000 0000 0000 0000 0000 0000  ................
```

> Note: `xxd -s <address>` starts displaying from the given address.

The section offset is 0x1829 (look at the section header `struct` in `man elf` or `elf.h` to understand the position of the offset value in the output above).

3) View the section at the offset 0x1b (should be start of string `.interp`):

```bash
$ xxd -s `python -c "print 0x1829 + 0x1b"` -l 10 c1
00001844: 2e69 6e74 6572 7000 2e6e                 .interp..n
```


## Dumping code from ELF files

### objdump

The classic way to disassemble files is `objdump -d <file>`, which dumps the instructions it finds in sections expected to contain code:

```bash
$ objdump -d c1

c1:     file format elf64-x86-64


Disassembly of section .init:

00000000004003c8 <_init>:
4003c8:       48 83 ec 08             sub    $0x8,%rsp
4003cc:       48 8b 05 25 0c 20 00    mov    0x200c25(%rip),%rax        # 600ff8 <__gmon_start__>
4003d3:       48 85 c0                test   %rax,%rax
...
```

### ndisasm

The NetWide Disassembler (you may know `nasm` - the NetWide Assembler) can be used to disassemble files and display the output in Intel syntax (as opposed to AT&T syntax, which is prevalent in Linux). The command needs a `-b <bit mode> parameter` (16, 32, 64)`:

```
$ ndisasm -b64 c1
00000000  7F45              jg 0x47
00000002  4C                rex.wr
00000003  460201            add r8b,[rcx]
00000006  0100              add [rax],eax
00000008  0000              add [rax],al
```

As you can see, by default disassembly is done from the start of the file. `ndisasm` is a simple disassembler which has no understanding of object file formats. It is best used for disassembling flat binary files (binary files with no headers) since it treats all bytes as instructions.

Try using the `-e` option with an offset, and `-o` with an address to see if you can disassemble the first few instructions of `.init` similarly to how `objdump` did above. Read `man ndisasm`.

> Note: `objdump` can also display Intel syntax with `objdump -M intel -d <file>`.

### Other

There are several more advanced tools which can be useful when doing disassembly and binary analyis. These include Ghidra, Radare2 and the Capstone framwork (which let's you build your own binary analyis tools in e.g. Python or C). We will look at some of these later.

Next, we'll cover linking.

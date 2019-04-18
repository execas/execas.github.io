---
layout: post
title: The ELF file format 3. Section headers.
date: 2019-04-18
tags: [c, linux, programming]
---


## Section headers

A section header contains information about a section, like its name, location in the file and type. An ELF file will have multiple section headers, organized in a *section header table*.

`<elf.h>` defines section headers for 32-bit and 64-bit binaries. The 64-bit section header is defined as follows:

```c
typedef struct
{
  Elf64_Word    sh_name;        /* Section name (string tbl index) */
  Elf64_Word    sh_type;        /* Section type */
  Elf64_Xword   sh_flags;       /* Section flags */
  Elf64_Addr    sh_addr;        /* Section virtual addr at execution */
  Elf64_Off sh_offset;          /* Section file offset */
  Elf64_Xword   sh_size;        /* Section size in bytes */
  Elf64_Word    sh_link;        /* Link to another section */
  Elf64_Word    sh_info;        /* Additional section information */
  Elf64_Xword   sh_addralign;   /* Section alignment */
  Elf64_Xword   sh_entsize;     /* Entry size if section holds table */
} Elf64_Shdr;
```

`sh_name` holds the index of the null-terminated string in section header string table (remember `e_shstrndx` in the file header).

`sh_type` holds the section type. Legal values are specified in `<elf.h>`:

```c
/* Legal values for sh_type (section type).  */

#define SHT_NULL           0            /* Section header table entry unused */
#define SHT_PROGBITS       1            /* Program data */
#define SHT_SYMTAB         2            /* Symbol table */
#define SHT_STRTAB         3            /* String table */
#define SHT_RELA           4            /* Relocation entries with addends */
#define SHT_HASH           5            /* Symbol hash table */
#define SHT_DYNAMIC        6            /* Dynamic linking information */
#define SHT_NOTE           7            /* Notes */
#define SHT_NOBITS         8            /* Program space with no data (bss) */
#define SHT_REL            9            /* Relocation entries, no addends */
#define SHT_SHLIB          10           /* Reserved */
#define SHT_DYNSYM         11           /* Dynamic linker symbol table */
#define SHT_INIT_ARRAY     14           /* Array of constructors */
#define SHT_FINI_ARRAY     15           /* Array of destructors */
#define SHT_PREINIT_ARRAY  16           /* Array of pre-constructors */
#define SHT_GROUP          17           /* Section group */
#define SHT_SYMTAB_SHNDX   18           /* Extended section indeces */
#define SHT_NUM            19           /* Number of defined types.  */
#define SHT_LOOS           0x60000000   /* Start OS-specific.  */
#define SHT_GNU_ATTRIBUTES 0x6ffffff5   /* Object attributes.  */
#define SHT_GNU_HASH       0x6ffffff6   /* GNU-style hash table.  */
#define SHT_GNU_LIBLIST    0x6ffffff7   /* Prelink library list */
#define SHT_CHECKSUM       0x6ffffff8   /* Checksum for DSO content.  */
#define SHT_LOSUNW         0x6ffffffa   /* Sun-specific low bound.  */
#define SHT_SUNW_move      0x6ffffffa
#define SHT_SUNW_COMDAT    0x6ffffffb
#define SHT_SUNW_syminfo   0x6ffffffc
#define SHT_GNU_verdef     0x6ffffffd   /* Version definition section.  */
#define SHT_GNU_verneed    0x6ffffffe   /* Version needs section.  */
#define SHT_GNU_versym     0x6fffffff   /* Version symbol table.  */
#define SHT_HISUNW         0x6fffffff   /* Sun-specific high bound.  */
#define SHT_HIOS           0x6fffffff   /* End OS-specific type */
#define SHT_LOPROC         0x70000000   /* Start of processor-specific */
#define SHT_HIPROC         0x7fffffff   /* End of processor-specific */
#define SHT_LOUSER         0x80000000   /* Start of application-specific */
#define SHT_HIUSER         0x8fffffff   /* End of application-specific */
```

`sh_flags` holds additional information about a section stored as flag bits.  Legal values are specified in `<elf.h>`:

```c
/* Legal values for sh_flags (section flags).  */

#define SHF_WRITE            (1 << 0)   /* Writable */
#define SHF_ALLOC            (1 << 1)   /* Occupies memory during execution */
#define SHF_EXECINSTR        (1 << 2)   /* Executable */
#define SHF_MERGE            (1 << 4)   /* Might be merged */
#define SHF_STRINGS          (1 << 5)   /* Contains nul-terminated strings */
#define SHF_INFO_LINK        (1 << 6)   /* `sh_info' contains SHT index */
#define SHF_LINK_ORDER       (1 << 7)   /* Preserve order after combining */
#define SHF_OS_NONCONFORMING (1 << 8)   /* Non-standard OS specific
                                           handling required */
#define SHF_GROUP            (1 << 9)   /* Section is member of a group.  */
#define SHF_TLS              (1 << 10)  /* Section hold thread-local data.  */
#define SHF_COMPRESSED       (1 << 11)  /* Section with compressed data. */
#define SHF_MASKOS           0x0ff00000 /* OS-specific.  */
#define SHF_MASKPROC         0xf0000000 /* Processor-specific */
#define SHF_ORDERED          (1 << 30)  /* Special ordering requirement
                                           (Solaris).  */
#define SHF_EXCLUDE          (1U << 31) /* Section is excluded unless
                                           referenced or allocated (Solaris).*/
```

`sh_addr` holds the address of the section's first byte if section appears in the memory image of a process, 0 otherwise.

`sh_offset` holds the byte offset from the beginning of the file to the first byte of the section.

`sh_size` holds the section's size in bytes.

`sh_link` and `sh_info` are used differently depending on the section type. The first holds an index to the section header table, while the latter holds "extra information". As an example, if the section type is `SHT_DYNAMIC`, `sh_link` will hold the index of the required string table, while `sh_info` will be 0.

`sh_addralign` holds a value for taking care of a section's alignment requirements. The section's address  (`sh_addr`) will be a multiple of this value.

`sh_entsize` holds the size of each entry if the section holds a table of fixed-size entries (like a symbol table). Set to 0 if section does not hold such a table.

### Understanding the ELF section header table

The start of the section headers (actually of the section header *table*) is specified in the file header:

<div class="term">
<b>~]$</b> readelf -h c1 | grep sec
  Start of section headers:          6408 (bytes into file)
  Size of section headers:           64 (bytes)
  Number of section headers:         31
</div>

The section header table is at the end of the ELF file, and is an array of section headers.

We can examine the section header table by dumping the hex with an offset to the beginning of the section headers (the section header table).

<div class="term">
<b>~]$</b> hexdump -C -s 6408 c1|head
00001908  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001948  1b 00 00 00 01 00 00 00  02 00 00 00 00 00 00 00  |................|
00001958  38 02 40 00 00 00 00 00  38 02 00 00 00 00 00 00  |8.@.....8.......|
00001968  1c 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001978  01 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001988  23 00 00 00 07 00 00 00  02 00 00 00 00 00 00 00  |#...............|
00001998  54 02 40 00 00 00 00 00  54 02 00 00 00 00 00 00  |T.@.....T.......|
000019a8  20 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  | ...............|
000019b8  04 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
</div>

Now, let's compare this to the output from `readelf`:

<div class="term">
<b>~]$</b> readelf -SW c1
There are 31 section headers, starting at offset 0x1908:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        0000000000400238 000238 00001c 00   A  0   0  1
  [ 2] .note.ABI-tag     NOTE            0000000000400254 000254 000020 00   A  0   0  4
  [ 3] .note.gnu.build-id NOTE            0000000000400274 000274 000024 00   A  0   0  4
  [ 4] .gnu.hash         GNU_HASH        0000000000400298 000298 00001c 00   A  5   0  8
  [ 5] .dynsym           DYNSYM          00000000004002b8 0002b8 000048 18   A  6   1  8
  [ 6] .dynstr           STRTAB          0000000000400300 000300 000038 00   A  0   0  1
  [ 7] .gnu.version      VERSYM          0000000000400338 000338 000006 02   A  5   0  2
  [ 8] .gnu.version_r    VERNEED         0000000000400340 000340 000020 00   A  6   1  8
  [ 9] .rela.dyn         RELA            0000000000400360 000360 000018 18   A  5   0  8
  [10] .rela.plt         RELA            0000000000400378 000378 000018 18  AI  5  24  8
  [11] .init             PROGBITS        0000000000400390 000390 00001a 00  AX  0   0  4
  [12] .plt              PROGBITS        00000000004003b0 0003b0 000020 10  AX  0   0 16
  [13] .plt.got          PROGBITS        00000000004003d0 0003d0 000008 00  AX  0   0  8
  [14] .text             PROGBITS        00000000004003e0 0003e0 000172 00  AX  0   0 16
  [15] .fini             PROGBITS        0000000000400554 000554 000009 00  AX  0   0  4
  [16] .rodata           PROGBITS        0000000000400560 000560 000010 00   A  0   0  8
  [17] .eh_frame_hdr     PROGBITS        0000000000400570 000570 000034 00   A  0   0  4
  [18] .eh_frame         PROGBITS        00000000004005a8 0005a8 0000f4 00   A  0   0  8
  [19] .init_array       INIT_ARRAY      0000000000600e10 000e10 000008 08  WA  0   0  8
  [20] .fini_array       FINI_ARRAY      0000000000600e18 000e18 000008 08  WA  0   0  8
  [21] .jcr              PROGBITS        0000000000600e20 000e20 000008 00  WA  0   0  8
  [22] .dynamic          DYNAMIC         0000000000600e28 000e28 0001d0 10  WA  6   0  8
  [23] .got              PROGBITS        0000000000600ff8 000ff8 000008 08  WA  0   0  8
  [24] .got.plt          PROGBITS        0000000000601000 001000 000020 08  WA  0   0  8
  [25] .data             PROGBITS        0000000000601020 001020 000004 00  WA  0   0  1
  [26] .bss              NOBITS          0000000000601024 001024 000004 00  WA  0   0  1
  [27] .comment          PROGBITS        0000000000000000 001024 00002d 01  MS  0   0  1
  [28] .symtab           SYMTAB          0000000000000000 001058 0005e8 18     29  47  8
  [29] .strtab           STRTAB          0000000000000000 001640 0001b5 00      0   0  1
  [30] .shstrtab         STRTAB          0000000000000000 0017f5 00010c 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
</div>

> Note `readelf -SW` is short for `readelf --section-headers --wide`.

All the data that is nicely presented in the `readelf` output, is available to us in the hexdump of the data, although it may take some extra time and some extra steps to extract it.

Since a section header is 64 bytes, that corresponds to 4 lines of hexdump output.

The first 4 lines are 0 (the asteriks hides three lines consisting of zeroes). All ELF files have a section header in index 0 of the section header table with all entries set to zero.

<div class="term">
00001908  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
</div>


For the next section header, let's try to decode it from the hex data using what we've learned:

<div class="term">
00001948  1b 00 00 00 01 00 00 00  02 00 00 00 00 00 00 00  |................|
00001958  38 02 40 00 00 00 00 00  38 02 00 00 00 00 00 00  |8.@.....8.......|
00001968  1c 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001978  01 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
</div>

The name, `sh_name` in `Elf64_Shdr`, is of type `Elf64_Word` (32 bits) and specifies the location of a null-terminated string in the section header string table (`.shstrtab`). The index of `.shstrtab` in the section header table is specified in `e_shstrndx in the ELF file header.

- The section header string table (`.shstrtab`) header is part of the section header table.
- The `e_shstrndx` in the file header specifies its index in the section header table.
- The value of `sh_name` specifies the location of the section name in `.shstrtab`.

**Finding the section header string table index**

- The file header is the 64 first bytes of the ELF file (see `e_ehsize` in the hexdump of the file header).
- The section header string table index (`_eshstrndx`) is the last 2 bytes of the file header (see `<elf.h>` file header structure).

<div class="term">
<b>~]$</b> hexdump -C -s 62 -n 2
0000003e  1e 00                                             |..|
00000040
</div>

001e is 30 in decimal. The `.shstrtab` section header has index 30 in the section header table.

> Note: We can more easily find this value using `readelf -h`

**Finding the section header string table**

- A section header is 64 bytes (see `e_shentsize` in the file header hexdump).
- Section headers are at the end of the ELF file.

The offset of the first section header is found at `e_shoff` in the file header):

<div class="term">
<b>~]$</b> hexdump -C -s 40 -n 8
00000028  08 19 00 00 00 00 00 00                           |........|
00000030
</div>

1908 is 6408 in decimal.

> Note: We can more easily find this value using `readelf -h`

Since every section header is 64 bytes, the `.shstrtab` header can be found using:

<div class="term">
<b>~]$</b> hexdump -C -s `python -c "print 6408 + (30*64)"` -n 64 c1
00002088  11 00 00 00 03 00 00 00  00 00 00 00 00 00 00 00  |................|
00002098  00 00 00 00 00 00 00 00  f5 17 00 00 00 00 00 00  |................|
000020a8  0c 01 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000020b8  01 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000020c8
</div>

The `.shrstrtab` header has the `sh_offset` entry which stores the offset of the section. These 8 bytes are in the second column of the second row: 00 00 00 00 00 00 17 f5.
The `sh_size` entry, 4 bytes, is found in the (first half of) the first column of the third row: 00 00 01 0c.

<div class="term">
<b>~]$</b> hexdump -C -s 0x17f5 c1 -n 0x010c
000017f5  00 2e 73 79 6d 74 61 62  00 2e 73 74 72 74 61 62  |..symtab..strtab|
00001805  00 2e 73 68 73 74 72 74  61 62 00 2e 69 6e 74 65  |..shstrtab..inte|
00001815  72 70 00 2e 6e 6f 74 65  2e 41 42 49 2d 74 61 67  |rp..note.ABI-tag|
00001825  00 2e 6e 6f 74 65 2e 67  6e 75 2e 62 75 69 6c 64  |..note.gnu.build|
00001835  2d 69 64 00 2e 67 6e 75  2e 68 61 73 68 00 2e 64  |-id..gnu.hash..d|
00001845  79 6e 73 79 6d 00 2e 64  79 6e 73 74 72 00 2e 67  |ynsym..dynstr..g|
00001855  6e 75 2e 76 65 72 73 69  6f 6e 00 2e 67 6e 75 2e  |nu.version..gnu.|
00001865  76 65 72 73 69 6f 6e 5f  72 00 2e 72 65 6c 61 2e  |version_r..rela.|
00001875  64 79 6e 00 2e 72 65 6c  61 2e 70 6c 74 00 2e 69  |dyn..rela.plt..i|
00001885  6e 69 74 00 2e 70 6c 74  2e 67 6f 74 00 2e 74 65  |nit..plt.got..te|
00001895  78 74 00 2e 66 69 6e 69  00 2e 72 6f 64 61 74 61  |xt..fini..rodata|
000018a5  00 2e 65 68 5f 66 72 61  6d 65 5f 68 64 72 00 2e  |..eh_frame_hdr..|
000018b5  65 68 5f 66 72 61 6d 65  00 2e 69 6e 69 74 5f 61  |eh_frame..init_a|
000018c5  72 72 61 79 00 2e 66 69  6e 69 5f 61 72 72 61 79  |rray..fini_array|
000018d5  00 2e 6a 63 72 00 2e 64  79 6e 61 6d 69 63 00 2e  |..jcr..dynamic..|
000018e5  67 6f 74 2e 70 6c 74 00  2e 64 61 74 61 00 2e 62  |got.plt..data..b|
000018f5  73 73 00 2e 63 6f 6d 6d  65 6e 74 00              |ss..comment.|
00001901
</div>

Now, to find the name, we count 0x1b bytes (see the `sh_name` data in the section header we are currently analyzing) into the section to find ".interp".

The remaining entries are also easily deciphered as long as you know the entries, the order of these in the data, and the amount of bytes reserved for each of them (see `<elf.h>`).

### Important section headers

| section | description |
|---------|-------------|
| `.interp` | pathname of the program interpreter|
| `.bss` |  uninitialized data|
| `.data`| initialized data|
| `.rodata`| read-only data|
| `.text`| executable instructions|
| `.init`/`.fini`| machine instructions for initialization/termination|
| `.symtab` and `.strtab` | symbol table and symbol table entries (strings)|
| `.dynsym` and `.dynstr` | dynamic linking symbol table and dynamic symbol table entries (strings)|
| `.shstrtab`| section header names (strings)|
| `.symtab`| symbol table|
| `.dynamic`| dynamic linking information|
| `.got`| global offset table|
| `.plt`| procedure linkage table|
| `.relNAME`| procedure linkage table|
| `.relaNAME`| procedure linkage table|

We can view the hexdump of a section using `readelf`:

<div class="term">
<b>~]$</b> readelf -x .init c1

Hex dump of section '.init':
  0x00400390 4883ec08 488b055d 0c200048 85c07405 H...H..]. .H..t.
  0x004003a0 e82b0000 004883c4 08c3              .+...H....

</div>

Sections containing machine instructions (like `.init`, `.fini` and `.text` can be more easily analyzed using `objdump` (note that the same hex data is still displayed):

<div class="term">
<b>~]$</b> objdump -j .init -d c1
c1:     file format elf64-x86-64


Disassembly of section .init:

0000000000400390 &lt;_init&gt;:
  400390:       48 83 ec 08             sub    $0x8,%rsp
  400394:       48 8b 05 5d 0c 20 00    mov    0x200c5d(%rip),%rax        # 600ff8 &lt;__gmon_start__&gt;
  40039b:       48 85 c0                test   %rax,%rax
  40039e:       74 05                   je     4003a5 <_init+0x15>
  4003a0:       e8 2b 00 00 00          callq  4003d0 <.plt.got>
  4003a5:       48 83 c4 08             add    $0x8,%rsp
  4003a9:       c3                      retq
</div>

> Note: `objdump -j <section> -d` is short for `objdump --section <section> --disassemble`.



---
layout: post
title: The ELF file format 2. The ELF file header
date: 2019-04-15
tags: [c, linux, programming]
---

## The ELF file header

The ELF file header appears at the start of every ELF file. It contains a lot of basic (but essential) information, and also serves as a sort of map to other important parts of the file.

`<elf.h>` defines file headers for 32-bit and 64-bit binaries. The 64-bit file header is defined as follows:

```c
#define EI_NIDENT (16)

typedef struct
{
    unsigned char e_ident[EI_NIDENT]; /* Magic number and other info */
    Elf64_Half  e_type;               /* Object file type */
    Elf64_Half  e_machine;            /* Architecture */
    Elf64_Word  e_version;            /* Object file version */
    Elf64_Addr  e_entry;              /* Entry point virtual address */
    Elf64_Off   e_phoff;              /* Program header table file offset */
    Elf64_Off   e_shoff;              /* Section header table file offset */
    Elf64_Word  e_flags;              /* Processor-specific flags */
    Elf64_Half  e_ehsize;             /* ELF header size in bytes */
    Elf64_Half  e_phentsize;          /* Program header table entry size */
    Elf64_Half  e_phnum;              /* Program header table entry count */
    Elf64_Half  e_shentsize;          /* Section header table entry size */
    Elf64_Half  e_shnum;              /* Section header table entry count */
    Elf64_Half  e_shstrndx;           /* Section header string table index */
} Elf64_Ehdr;
```

### Understanding the ELF file header

Compilation of the code below will produce an ELF 64-bit binary on an x86_64 Linux machine that we will examine to better understand the ELF file header.

```c
/*
 * c1.c
 * gcc c1.c -o c1 -std=c99
 */

int main() {}
```

We can use `readelf` to examine the file header of this binary:

```bash 
$ readelf -h c1
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x4003e0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          6408 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         31
  Section header string table index: 30
```

> Note: `-h` is the short option of `--file-header`

The **'Magic'** row corresponds to the `unsigned char e_ident[16]` array in the above file header structure.

ELF files always begin with the bytes "7f 45 4c 46", as we can see in `<elf.h>` in the definitions following the file header structs:

```c
/* Fields in the e_ident array.  The EI_* macros are indices into the
   array.  The macros under each EI_* macro are the values the byte
   may have.  */

#define EI_MAG0     0       /* File identification byte 0 index */
#define ELFMAG0     0x7f    /* Magic number byte 0 */

#define EI_MAG1     1       /* File identification byte 1 index */
#define ELFMAG1     'E'     /* Magic number byte 1 */

#define EI_MAG2     2       /* File identification byte 2 index */
#define ELFMAG2     'L'     /* Magic number byte 2 */

#define EI_MAG3     3       /* File identification byte 3 index */
#define ELFMAG3     'F'     /* Magic number byte 3 */
```

Next, **'Class'**, **'Data'**, **'Version'**, **'OS/ABI'** and **'ABI VERSION'** in the `readelf` output are also part of `e_ident` array.

> Note: The value of these can be seen in the `readelf` output's **'Magic'** row, as well as on their dedicated rows.

**'Class'** has position 4 in the array, and can be 0-3:

```c
#define EI_CLASS        4      /* File class byte index */
#define ELFCLASSNONE    0      /* Invalid class */
#define ELFCLASS32      1      /* 32-bit objects */
#define ELFCLASS64      2      /* 64-bit objects */
#define ELFCLASSNUM     3
```

**'Data'** has position 5 in the array, and can be 0-3:

```c
#define EI_DATA         5      /* Data encoding byte index */
#define ELFDATANONE     0      /* Invalid data encoding */
#define ELFDATA2LSB     1      /* 2's complement, little endian */
#define ELFDATA2MSB     2      /* 2's complement, big endian */
#define ELFDATANUM      3
```

**'Version'** has position 6 in the array, and must be `EV_CURRENT`:

```c
#define EI_VERSION  6          /* File version byte index */
                               /* Value must be EV_CURRENT */
```

`EV_CURRENT` is defined farther down in `<elf.h>`:

```c
/* Legal values for e_version (version).  */

#define EV_NONE     0          /* Invalid ELF version */
#define EV_CURRENT  1          /* Current version */
#define EV_NUM      2
```

**'OS/ABI'** has position 7 in the array, and can have a lot of different values:

```c
#define EI_OSABI            7       /* OS ABI identification */
#define ELFOSABI_NONE       0       /* UNIX System V ABI */
#define ELFOSABI_SYSV       0       /* Alias.  */
#define ELFOSABI_HPUX       1       /* HP-UX */
#define ELFOSABI_NETBSD     2       /* NetBSD.  */
#define ELFOSABI_GNU        3       /* Object uses GNU ELF extensions.  */
#define ELFOSABI_LINUX      ELFOSABI_GNU /* Compatibility alias.  */
#define ELFOSABI_SOLARIS    6       /* Sun Solaris.  */
#define ELFOSABI_AIX        7       /* IBM AIX.  */
#define ELFOSABI_IRIX       8       /* SGI Irix.  */
#define ELFOSABI_FREEBSD    9       /* FreeBSD.  */
#define ELFOSABI_TRU64      10      /* Compaq TRU64 UNIX.  */
#define ELFOSABI_MODESTO    11      /* Novell Modesto.  */
#define ELFOSABI_OPENBSD    12      /* OpenBSD.  */
#define ELFOSABI_ARM_AEABI  64      /* ARM EABI */
#define ELFOSABI_ARM        97      /* ARM */
#define ELFOSABI_STANDALONE 255     /* Standalone (embedded) application */
```

> Note: ABI stands for Application Binary Interface.

**'ABI Version'** has position 8 in the array, and is typically set to 0 by applications:

```c
#define EI_ABIVERSION   8   /* ABI version */
```

Lastly, we have `EI_PAD`, which occupies the rest of the bytes of the `e_ident` array. These bytes are not in use (reserved for future use) and are set to 0.

```c
#define EI_PAD      9       /* Byte index of padding bytes */
```

Following the `e_ident` array, we have:

`e_type` (**'Type'** in the `readelf` output). This identifies the object file type as one of the following:

```c
/* Legal values for e_type (object file type).  */

#define ET_NONE     0       /* No file type */
#define ET_REL      1       /* Relocatable file */
#define ET_EXEC     2       /* Executable file */
#define ET_DYN      3       /* Shared object file */
#define ET_CORE     4       /* Core file */
#define ET_NUM      5       /* Number of defined types */
#define ET_LOOS     0xfe00  /* OS-specific range start */
#define ET_HIOS     0xfeff  /* OS-specific range end */
#define ET_LOPROC   0xff00  /* Processor-specific range start */
#define ET_HIPROC   0xffff  /* Processor-specific range end */
```

`e_machine` (**'Machine'** in the `readelf` output). This specifies the required architecture for an individual file:

```c
#define EM_NONE      0  /* No machine */
#define EM_M32       1  /* AT&T WE 32100 */
#define EM_SPARC     2  /* SUN SPARC */
#define EM_386       3  /* Intel 80386 */
#define EM_68K       4  /* Motorola m68k family */
#define EM_88K       5  /* Motorola m88k family */
#define EM_IAMCU     6  /* Intel MCU */
#define EM_860       7  /* Intel 80860 */
...
...
...
```

The rest of the entries are described in the below table:

|entry | readelf output | description
|----- |------------|-------------|
|`e_version` |**'Version'** | Discussed earlier.|
|`e_entry` |**'Entry point address'** | Specifies the virtual address the system transfers control to to begin execution of the program.|
|`e_phoff` |**'Start of program headers'** | Holds the program header table's offset in bytes. 0 if no prgoram header table.|
|`e_shoff` |**'Start of section headers'** | Holds the section header table's offset in bytes. 0 if no section header table.|
|`e_flags` |**'Flags'** | Holds processor-specific flags.|
|`e_ehsize` |**'Size of this header'** | Holds the size of the ELF header in bytes.|
|`e_phentsize` |**'Size of program headers'** | Holds the size of one program header table entry in bytes. All entries are of same size.|
|`e_phnum` |**'Number of program headers'** | Holds the number of program header table entries.|
|`e_shentsize` |**'Size of section headers'**| Holds the size of one section header table entry in bytes. All entries are of same size.|
|`e_shnum` |**'Number of section headers'** | Holds the number of section header table entries.|
|`e_shstrndx` |**'Section header string table index'** | Holds the section header string table index.|

### The file header in a sea of bytes

Now that we have looked at all the members of the `Elf64_Ehdr`, we can try to use what we've learned to recognize the header in a sea of bytes.

The file header is at the beginning of a binary and begins with the magic bytes:

```bash 
$ hexdump -C c1|head
00000000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010  02 00 3e 00 01 00 00 00  e0 03 40 00 00 00 00 00  |..>.......@.....|
00000020  40 00 00 00 00 00 00 00  08 19 00 00 00 00 00 00  |@...............|
00000030  00 00 00 00 40 00 38 00  09 00 40 00 1f 00 1e 00  |....@.8...@.....|
00000040  06 00 00 00 05 00 00 00  40 00 00 00 00 00 00 00  |........@.......|
00000050  40 00 40 00 00 00 00 00  40 00 40 00 00 00 00 00  |@.@.....@.@.....|
00000060  f8 01 00 00 00 00 00 00  f8 01 00 00 00 00 00 00  |................|
00000070  08 00 00 00 00 00 00 00  03 00 00 00 04 00 00 00  |................|
00000080  38 02 00 00 00 00 00 00  38 02 40 00 00 00 00 00  |8.......8.@.....|
00000090  38 02 40 00 00 00 00 00  1c 00 00 00 00 00 00 00  |8.@.............|
```

The first row is the 16 bytes of the `e_ident` array:

- 7f 'E' 'L' 'F' (Magic bytes)
- 02 (class is ELF64)
- 01 (data encoding is 2's complement, little endian)
- 01 (version is current)
- 00 (OS/ABI is UNIX - System V)
- 00 (ABI version is 0)
- the remaining 7 bytes is padding

`e_type` is next, type `Elf64_Half` (16 bits):

- 02 00 (remember, Little-endian) is 2, which means *executable file*.

`e_machine`, type `Elf64_Half`:

- 3e 00 is 62, which means *AMD x86-64 architecture*.

`e_version`, type Elf64_Word (32 bits):

- 01 00 00 00 is 1, meaning *current version*.

`e_entry`, type Elf64_Addr (64 bits):

- e0 03 40 00 00 00 00 00 00 is the address 0x4003e0.

These are followed by the remaining members of the ELF file header, and after those we are entering program header territory, but let's look at section headers first, since they are the next part of `<elf.h>`.

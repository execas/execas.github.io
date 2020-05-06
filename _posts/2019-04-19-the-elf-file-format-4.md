---
layout: post
title: The ELF file format 4. Program headers
date: 2019-04-19
tags: [c, linux, programming]
---

As covered in the previous post, *section* headers contain information about *sections*. *Program* headers contain information about *segments*. Segments often contain one or more sections. Sections are the view used for linking and relocation, segments are the view used for execution (by the OS and dynamic linker).

`<elf.h>` defines progam headers for 32-bit and 64-bit binaries. The 64-bit program header is defined as follows:

```c
typedef struct
{
  Elf64_Word	p_type;			/* Segment type */
  Elf64_Word	p_flags;		/* Segment flags */
  Elf64_Off	    p_offset;		/* Segment file offset */
  Elf64_Addr	p_vaddr;		/* Segment virtual address */
  Elf64_Addr	p_paddr;		/* Segment physical address */
  Elf64_Xword	p_filesz;		/* Segment size in file */
  Elf64_Xword	p_memsz;		/* Segment size in memory */
  Elf64_Xword	p_align;		/* Segment alignment */
} Elf64_Phdr;
```

`p_type` describes the segment type and can have the following values:

```c
/* Legal values for p_type (segment type).  */

#define	PT_NULL		0		/* Program header table entry unused */
#define PT_LOAD		1		/* Loadable program segment */
#define PT_DYNAMIC	2		/* Dynamic linking information */
#define PT_INTERP	3		/* Program interpreter */
#define PT_NOTE		4		/* Auxiliary information */
#define PT_SHLIB	5		/* Reserved */
#define PT_PHDR		6		/* Entry for header table itself */
#define PT_TLS		7		/* Thread-local storage segment */
#define	PT_NUM		8		/* Number of defined types */
#define PT_LOOS		0x60000000	/* Start of OS-specific */
#define PT_GNU_EH_FRAME	0x6474e550	/* GCC .eh_frame_hdr segment */
#define PT_GNU_STACK	0x6474e551	/* Indicates stack executability */
#define PT_GNU_RELRO	0x6474e552	/* Read-only after relocation */
#define PT_LOSUNW	0x6ffffffa
#define PT_SUNWBSS	0x6ffffffa	/* Sun Specific segment */
#define PT_SUNWSTACK	0x6ffffffb	/* Stack segment */
#define PT_HISUNW	0x6fffffff
#define PT_HIOS		0x6fffffff	/* End of OS-specific */
#define PT_LOPROC	0x70000000	/* Start of processor-specific */
#define PT_HIPROC	0x7fffffff	/* End of processor-specific */
```

`p_flags` holds the segment's flags. As an example, a text segment will typically be executable and readable, while a data segment may be readable and writeable. The values are defined in `<elf.h>`:

```c
#define PF_X		(1 << 0)	/* Segment is executable */
#define PF_W		(1 << 1)	/* Segment is writable */
#define PF_R		(1 << 2)	/* Segment is readable */
#define PF_MASKOS	0x0ff00000	/* OS-specific */
#define PF_MASKPROC	0xf0000000	/* Processor-specific */
```

`p_offset` is the offset to the segment from the beginning of the file.

`p_vaddr` is the virtual address of the segment's first byte in memory.

`p_paddr` is used on systems where physical addressing is relevant.

`p_filesz` is the number of bytes in the file image of the segment.

`p_memsz` is the number of bytes in the memory image of the segment.

`p_align` holds a value for taking care of the segment's alignment requirements.


## Understanding the ELF program header table

The start of the program headers (actually of the program header *table*) is specified in the file header:

```bash
$ readelf -h c1 | grep prog
Start of program headers:          64 (bytes into file)
Size of program headers:           56 (bytes)
Number of program headers:         9
```

As with sections headers, we could use this information to read out the bytes using a tool like `xxd` or `hexdump`, but this is left as an exercise for the reader. We will instead look at how `readelf` displays this information:

```bash
$ readelf -lW c1

Elf file type is EXEC (Executable file)
Entry point 0x400430
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  PHDR           0x000040 0x0000000000400040 0x0000000000400040 0x0001f8 0x0001f8 R E 0x8
  INTERP         0x000238 0x0000000000400238 0x0000000000400238 0x00001c 0x00001c R   0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x000000 0x0000000000400000 0x0000000000400000 0x000714 0x000714 R E 0x200000
  LOAD           0x000e10 0x0000000000600e10 0x0000000000600e10 0x00021c 0x000220 RW  0x200000
  DYNAMIC        0x000e28 0x0000000000600e28 0x0000000000600e28 0x0001d0 0x0001d0 RW  0x8
  NOTE           0x000254 0x0000000000400254 0x0000000000400254 0x000044 0x000044 R   0x4
  GNU_EH_FRAME   0x0005f4 0x00000000004005f4 0x00000000004005f4 0x000034 0x000034 R   0x4
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10
  GNU_RELRO      0x000e10 0x0000000000600e10 0x0000000000600e10 0x0001f0 0x0001f0 R   0x1

 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .plt.got .text .fini .rodata .eh_frame_hdr .eh_frame
   03     .init_array .fini_array .jcr .dynamic .got .got.plt .data .bss
   04     .dynamic
   05     .note.ABI-tag .note.gnu.build-id
   06     .eh_frame_hdr
   07
   08     .init_array .fini_array .jcr .dynamic .got
```

> Note: `readelf -lW` is short for `readelf --program-headers --wide`.

First, we get information about the file type and entry point, as well as the number of progam headers and their starting offset.

Next follows the program headers, and all the information we saw as part of the `Elf64_Phdr` structure is displayed here (from left to right): `p_type`, `p_offset`, `p_vaddr`, `p_paddr`, `p_filesz`, `p_memsz`, `p_flags` and `p_align`. Note for example that the stack is readable and writeable, but not executable.

At the end of the output listing is a section to segment mapping. Note that the number corresponds to a program header printed above, e.g. 07 is the `GNU_STACK` header. Most segments contain one or more sections, but it may not be clear how this mapping is done.

The answer is really simple. Look at the address and offset of for example 02. Note the address and offset. View the section headers by using `readelf -SW`. Note that `.interp` is the first and `.eh_frame` is the last section in the range defined by the segment's address and offset. Look at another segment and confirm the same.

Next, we'll look at the various tools and commands we can use to study and make sense of ELF files.

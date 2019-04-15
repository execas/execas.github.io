---
layout: post
title: The ELF file format 1: Introduction and the basic types
date: 2019-04-15
tags: [c, linux, programming]
---

In this series of articles we will look at the ELF file format by studying the `<elf.h>` header file in detail.

## Introduction

The ELF file format in `<elf.h>` defines the format of ELF executable binary files.

> Note: ELF stands for *Executable and Linkable format*.

The ELF format is the standard binary file format for Linux, and is used for executable files, object code, core dumps and shared libraries.

An ELF file looks something like this:

```
    ===========================
    |    ELF file header      |
    |-------------------------|
    |  program header table   |
    |-------------------------|
    |                         | \
    |                         |  \
                ...                data referred to by the tables
    |                         |  /
    |                         | /
    |-------------------------|
    |  section header table   |
    ===========================
```


### The first lines of code

All the code in `<elf.h> is defined inside an **#include guard** to prevent double inclusion:

```
#ifndef _ELF_H
#define _ELF_H 1

#include <features.h>

__BEGIN_DECLS

(((ALL CODE)))

__END_DECLS

#endif
```

The first line includes `<features.h>`, a Linux/glibc-specific header file containing feature test macros that "allow the programmer to control the definitions that are exposed by system header files when a program is compiled".

See `$ man feature_test_macros` to learn more.

Next we got `__BEGIN_DECLS` (and `__END_DECLS`), which are defined as follows in `<sys/cdefs.h>` (included by `<features.h`>):

```
/* C++ needs to know that types and declarations are C, not C++.  */
#ifdef  __cplusplus
# define __BEGIN_DECLS  extern "C" {
# define __END_DECLS    }
#else
# define __BEGIN_DECLS
# define __END_DECLS
#endif
```

The next lines of code defines the standard ELF types.

## Standard ELF Types

The standard ELF types are aliases for the specific width types found in `<stdint.h>`:

```c
/* Standard ELF types.  */

#include <stdint.h>

/* Type for a 16-bit quantity.  */
typedef uint16_t Elf32_Half;
typedef uint16_t Elf64_Half;

/* Types for signed and unsigned 32-bit quantities.  */
typedef uint32_t Elf32_Word;
typedef int32_t  Elf32_Sword;
typedef uint32_t Elf64_Word;
typedef int32_t  Elf64_Sword;

/* Types for signed and unsigned 64-bit quantities.  */
typedef uint64_t Elf32_Xword;
typedef int64_t  Elf32_Sxword;
typedef uint64_t Elf64_Xword;
typedef int64_t  Elf64_Sxword;

/* Type of addresses.  */
typedef uint32_t Elf32_Addr;
typedef uint64_t Elf64_Addr;

/* Type of file offsets.  */
typedef uint32_t Elf32_Off;
typedef uint64_t Elf64_Off;

/* Type for section indices, which are 16-bit quantities.  */
typedef uint16_t Elf32_Section;
typedef uint16_t Elf64_Section;

/* Type for version symbol information.  */
typedef Elf32_Half Elf32_Versym;
typedef Elf64_Half Elf64_Versym;
```

These types are used throughout the entire `<elf.h>`, so we will have them properly memorized after referring back to the above definitions numerous times.

Next, we will look at the ELF file header.

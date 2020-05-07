---
layout: post
title: Finding files based on timestamps
date: 2018-10-30
tags: [find, security, linux, tools]
---

## Timestamp file search

### Access (`atime`)

The *access* timestamp is updated when a file is opened for reading or writing.
On a file system mounted with `relatime` (part of `defaults`, which is very common), `atime` will update on access if the file has been modified or changed after it was last accessed, or if the previous `atime` update was more than 24 hours ago.

Create files with various access times for testing purposes:

```bash
$ mkdir test; cd test
$ touch -a -d "2 days ago" 2da
$ touch -a -d "10 days ago" 10da
$ touch -a -d "30 min ago" 30ma
```

Find files accessed more than 2 days ago:

```bash
$ find -atime +2
./10da
```

Find files accessed less than 2 days ago:

```bash
$ find -atime -2
./30ma
```

Calling the argument to `find -atime` *days* is not really correct, as the command figures out how long ago the file was accessed in *periods* of 24 hours, and ignores any fractional part.
The file "2da" has an `atime` of exactly 2 periods of 24 hours:

```bash
find -atime 2
./2da
```

Find a file accessed less than 100 minutes ago:

```bash
$ find -amin -100
./30ma
```

### Modify (`mtime`) and Change (`ctime`)

The *modify* timestamp is updated when a file's content is modified.
The *change* timestamp is updated when a file's metadata is changed.

The options `-mtime <n>` and `-ctime <n>` work in the same way as `-atime <n>`, but find files based on modification time and change time, respectively.

The options `-mmin <n>` and `-cmin <n>` are similar to `-amin <n>`.

## The `-newer` options

### `-newer`

This option let's you find files that have been accessed, modified or changed more recently than a specified file was *modified*.

|option          | finds
|----------------|------------------------------------------------|
|`-newer <file>` | files *modified* after `<file>` was modified|
|`-cnewer <file>` |files *changed* after `<file>` was modified|
|`-anewer <file>` | files *accessed* after `<file>` was modified|

### `-newerXY`

This options let's you find files by comparing specified timestamps with those of a reference. The reference can be a file or a string describing an absolute time.

The command has the syntax `find -newerXY <reference>`. `X` and `Y` are placeholders:

 - `X` is a/m/c to specify if `find` should look at atime/mtime/ctime of the files it searches through.
 - `Y` is a/m/c to specify which timestamp to use from the file reference, or `-t` if you specify an abosulute time.


Find files modified, changed or accessed after specifed date:

```bash
$ find -newermt 2016-09-04       ○ X is m(time), Y is (t)ime, ref is time-string
$ find -newerct yesterday        ○ X is c(time), Y is (t)ime, ref is time-string
$ find -newerat "last friday"    ○ X is a(time), Y is (t)ime, ref is time-string
```

Find files modified, changed or accessed after a file reference was changed:

```bash
$ find -newermc foo              ○ X is m(time), Y is c(time), ref is file
$ find -newercc foo              ○ X is c(time), Y is c(time), ref is file
$ find -newerac foo              ○ X is a(time), Y is c(time), ref is file
```

---
layout: post
title: The Linux find command
date: 2018-09-07
tags: [find, linux, tools]
---

The `find` command is a very powerful command line tool to search for files in a directory hierarchy.

## Basic usage

Find file or directory named 'query' in the current dir or subdirs:

```bash
$ find -name "query"
./query
$ find "query"
./query
```
Same as above, but ignore case:

```bash
$ find -iname "query"
./query
./Query
```
Find a file or directory named 'cat' in `/usr/bin` or subdirs.

```bash
$ find /usr/bin -name "cat"
/usr/bin/cat
```

### Finding specific types

- (b)lock
- (c)haracter
- (d)ir
- (f)ile
- (l)ink, symbolic
- (p)ipe
- (s)ocket

```bash
find  -type [bcdflps]
```

### Limiting the search depth

To limit the levels of subdirs searched, use `-maxdepth <n>`.

## Some examples

Find all non-hidden directories in current dir:

```bash
$ find -type d -maxdepth 1 -not -name ".*"
$ find -type d -maxdepth 1 ! -name ".*"
```

Find SUID set files in root dir:

```bash
$ find / -perm /u=s
```

Delete all empty txt files:

```bash
$ find -type f -empty -name "*.txt" -exec rm -f '{}' \;
```

Find all files **not** belonging to a specific user:

```bash
$ find -type f ! -user cas
```

## Other important options

### The execute options

With `-exec ls '{}' \;` the `ls` command is executed for each file, so:

```bash
$ find -exec ls '{}' \;
```

will execute:

```bash
ls a.txt
ls b.txt
ls c.txt
...
```

The `-exec ls '{}' \+` is more efficient as a lot of filenames are supplied as arguments to the command at once, preventing it being called once for each file:

```
$ ls a.txt b.txt c.txt ...
```

An alternative is to use `xargs`, which is similar to `-exec ls '{}' \+`:

```bash
$ find . -name "*.txt" | xargs cat
```

`-ok` does not support `\+`. User is asked before command execution for each match.

```bash
$ find -ok rm -f '{}' \;
```

The `-execdir` option has the same choice of postfix (`\;` or `\+`), and works the same as `-exec` execept that the command is executed from the subdirectory containing the matched file. There is also an `-okdir`.


### Print

`-print`, printing matches followed by a newline to the standard output, is the default action of `find`.
When other actions are used, like for example `-exec`, matches are not printed, and `-print` must be explicitly specified.

```bash
$ find . -exec file '{}' \+ -print
```

`-print0` uses a null character instead of a newline.


### Size

Find files above 2 mB:

```
$ find . -size +2M
```

Find files below 2 gB:

```
$ find . -size -2G
```

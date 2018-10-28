---
layout: post
title: Finding files based on ownership and permissions
date: 2018-10-28
tags: [find, security, linux, tools]
---

## Finding files based on ownership

Find all files belonging to a specified user:

```bash
$ find /home/dana/shared -user henry
```
Find all files with a specified user ID.

```bash
$ find /home/dana/shared -uid 1077
```

Find all files *not* belonging to a specified group:

```bash
$ find /usr -not -group root
/usr/bin/wall
/usr/bin/write
/usr/bin/ssh-agent
...
```

## Finding files based on permissions


### Test setup

```bash
$ mkdir test; cd test             ○ Create test directory, go into it
$ touch 222 644 700 755 777       ○ Create dirs
$ chmod 222 222                   ○ Set permissions
$ chmod 644 644
$ chmod 700 700
$ chmod 755 755
$ chmod 777 777
```

### `find -perm` syntax

Find files with permission *exactly* 777:

```bash
# find -perm 777
./777
```

Find files where a specified somebody has a specified permission (possibly in addition to any other permissions):

```bash
$ find -perm /u=x
.
./700
./755
./777
$ find -perm /g=w
.
./222
./777
```

Find files with permissions *at least* 644 (owner has at least read and write, group owner and others have at least write):

```bash
$ find -perm -644
.
./644
./755
./777
```

Find files where *any* of UGO are set to readable :

```bash
$ find -perm /444
./644
./700
./755
./777
```

### Other options

Find executable, writable and readable files, but take into account ACL's and other permission artefacts which `-perm` ignores:

```bash
$ find -executable -type f
$ find -readable -type f
$ find -writable -type f
```


### Find setuid, setgid and sticky bit


Find setuid files, list with permissions:

```bash
$ find /usr/bin -perm /u=s | xargs ls -l
-rwsr-xr-x. 1 root root      52952 Apr 11  2018 /usr/bin/at
-rwsr-xr-x. 1 root root      64240 Nov  5  2016 /usr/bin/chage
...
```

Find setgid files, list with permissions:

```bash
$ find /usr/bin -perm /g=s -type f | xargs ls -l
-rwxr-sr-x. 1 root cgred    15624 Apr 11  2018 /usr/bin/cgclassify
-rwxr-sr-x. 1 root cgred    15592 Apr 11  2018 /usr/bin/cgexec
-rwx--s--x. 1 root slocate  40520 Apr 11  2018 /usr/bin/locate
...
```

Find setgid directories:

```bash
# find / -perm /g=s -type d
/run/log/journal
```

Find directories with sticky bit:

```bash
find / -perm /o=t
...
```

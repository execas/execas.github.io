---
layout: post
title: "Linux file size"
date: 2018-09-08
tags: [linux, find]
---

## The size of things


### `ls`

List files, display total size and size of each file in human-readable format (`-h`):

```bash
$ ls -lh
total 1.1G
-rw-rw-r--. 1 user user 1.1G Sep   6  19:52 big
-rw-rw-r--. 1 user user  20M Sep   6  19:53 med
drwxrwxr-x. 1 user user    6 Sep   6  19:54 sd
-rw-rw-r--. 1 user user 1.0M Sep   6  19:50 small
-rw-rw-r--. 1 user user  50K Sep   6  19:49 xs
```

### `du`

Display the size of the working directory and subdirs, human-readable format:

```bash
$ du -h
0    ./sd
1.1G .
```

Display the size of a specfied directory and subdirs:

```bash
$ du /usr/bin
115304   /usr/bin
```

Display the human-readable size of the working directory and subdirs, with a total at the end:

```bash
$ du -hc
0    ./sd
1.1G .
1.1G total
```

Display the total size of the working directory with subdirs:

```bash
$ du -hs
1.1G .
```


## Finding files based on size

The Linux `find` command's `-size` option an be used with the following units:

|    |                                                           |
|----|-----------------------------------------------------------|
|b | 512-byte blocks (this is the default if no suffix is used)|
|c | bytes
|w | two-byte words
|k | Kilobytes (units of 1024 bytes)
|M | Megabytes (units of 1048576 bytes)
|G | Gigabytes (units of 1073741824 bytes)




Find files above 2 mB:

```
$ find . -size +2M
```

Find files below 2 gB:

```
$ find . -size -2G
```

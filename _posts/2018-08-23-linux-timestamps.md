---
layout: post
title: Linux timestamps
date: 2018-08-23
tags: [linux,security]
---

Files and dirs have three timestamps in Linux: Access, change and modify.
These are often referred to as `atime`, `ctime` and `mtime`.


## Reading time stamps

All of these timestamps can be read with `stat`:

```bash
$ echo hello > foo
$ stat foo
...
Access: 2018-08-23 21:11:16.685201987 +0200
Modify: 2018-08-23 21:11:16.685201987 +0200
Change: 2018-08-23 21:11:16.685201987 +0200
```
Note that for the newly created file, all timestamps are equal.

## Updating time stamps

`touch` will update **all** timestamps to the current time.

### Updating `atime`

```bash
$ cat foo
hello
$ stat foo
...
Access: 2018-08-23 21:12:26
Modify: 2018-08-23 21:11:16
Change: 2018-08-23 21:11:16
```

Notice how the only the access time stamp has changed.
Opening the file for reading or writing will update `atime`.

Example commands that will change `atime`:

- grep
- vim
- less

### Updating `ctime`

```bash
$ chmod 700 foo
$ stat foo
...
Access: 2018-08-23 21:12:26
Modify: 2018-08-23 21:11:16
Change: 2018-08-23 21:13:56
```

Notice how only the change time stamp has changed.
Changing the file's metadata will update `ctime`.

Example commands that will change `atime`:

- mv
- chown
- chattr

### Updating `mtime`

```bash
$ echo "hello" > foo
$ stat foo
...
Access: 2018-08-23 21:12:26
Modify: 2018-08-23 21:15:16
Change: 2018-08-23 21:15:16
```

Notice how both the modify and the change time stamps have been updated.
Modifying the file's contents will update `mtime` and `ctime`.

## Manually setting time stamps

### `atime` and `mtime`

```
$ touch test
$ stat test
...
Access: 2018-08-23 21:52:37
Modify: 2018-08-23 21:52:37
Change: 2018-08-23 21:52:37
$ touch -d "6 hours ago" test
$ stat test
...
Access: 2018-08-23 15:52:37
Modify: 2018-08-23 15:52:37
Change: 2018-08-23 21:52:37
```

### `atime` only

```
$ touch test
$ stat test
...
Access: 2018-08-24 22:24:21
Modify: 2018-08-24 22:24:21
Change: 2018-08-24 22:24:21
$ touch -a -d "last monday"
$ stat test
...
Access: 2018-08-20 00:00:00
Modify: 2018-08-24 22:24:21
Change: 2018-08-24 22:25:28
```

### `mtime` only

```
$ touch test
$ stat test
...
Access: 2018-08-24 22:26:59
Modify: 2018-08-24 22:26:59
Change: 2018-08-24 22:26:59
$ touch -m -t 201801061830.09 test
...
Access: 2018-08-24 22:26:59
Modify: 2018-01-06 18:30:09
Change: 2018-08-24 22:28:49
```

### `ctime` only

Change system time as root, then change the file, e.g. with `touch`.

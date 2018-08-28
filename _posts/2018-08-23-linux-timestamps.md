---
layout: post
title: Linux timestamps
date: 2018-08-23
tags: [linux,security]
---

Files and dirs have three timestamps in Linux: *Access*, *Change* and *Modify*.
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

Notice how only the *access* time stamp has been updated.
Opening the file for reading or writing will update `atime`.


> If the filesystem is mounted with `defaults`, `relatime` is used.
>
> With `relatime`, `atime` will update on access if its value is less than or equal to `mtime` and `ctime`, and also if the previous `atime` update was more than 24 hours ago.

Example commands that will update `atime`:

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

Notice how only the *change* time stamp has been updated.
Changing the file's metadata will update `ctime`.

Example commands that will update `atime`:

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

Notice how both the *modify* and the *change* time stamps have been updated.
Modifying the file's contents will update `mtime` and `ctime`.

## Directory timestamps

### Creating a directory

```
$ mkdir dir
$ stat dir
...
Access: 2018-08-23 23:02:40
Modify: 2018-08-23 23:02:40
Change: 2018-08-23 23:02:40
```

### Creating a file in a directory

```
$ touch dir/foo
$ stat dir
...
Access: 2018-08-23 23:02:40
Modify: 2018-08-23 23:04:48
Change: 2018-08-23 23:04:48
```

Notice that both `mtime` and `ctime` has changed.

### Directory `atime`

Create dir:

```bash
$ mkdir dir
$ stat dir
Access: 2018-08-23 23:05:00
Modify: 2018-08-23 23:05:00
Change: 2018-08-23 23:05:00
```

List dir:

```bash
$ ls
$ stat dir
Access: 2018-08-23 23:06:55
Modify: 2018-08-23 23:05:00
Change: 2018-08-23 23:05:00
```

`atime` is updated. List dir again:

```bash
$ ls
$ stat dir
Access: 2018-08-23 23:06:55
Modify: 2018-08-23 23:05:00
Change: 2018-08-23 23:05:00
```

`atime` is the same. Create a file:

```bash
$ touch dir/foo
$ stat dir
Access: 2018-08-23 23:06:55
Modify: 2018-08-23 23:07:10
Change: 2018-08-23 23:07:10
```

`ctime` and `mtime` are updated. List dir again:


```bash
$ ls dir
$ stat dir
Access: 2018-08-23 23:09:33
Modify: 2018-08-23 23:07:10
Change: 2018-08-23 23:07:10
```

`atime` is updated.



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
Access: 2018-08-23 22:24:21
Modify: 2018-08-23 22:24:21
Change: 2018-08-23 22:24:21
$ touch -a -d "last monday"
$ stat test
...
Access: 2018-08-20 00:00:00
Modify: 2018-08-23 22:24:21
Change: 2018-08-23 22:25:28
```

### `mtime` only

```
$ touch test
$ stat test
...
Access: 2018-08-23 22:26:59
Modify: 2018-08-23 22:26:59
Change: 2018-08-23 22:26:59
$ touch -m -t 201801061830.09 test
...
Access: 2018-08-23 22:26:59
Modify: 2018-01-06 18:30:09
Change: 2018-08-23 22:28:49
```

Notice how `mtime` is set manually, while `ctime` is updated to the current time.

### `ctime` only

Change system time as root, then change the file, e.g. with `touch`.

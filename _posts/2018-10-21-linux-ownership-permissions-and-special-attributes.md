---
layout: post
title: Linux ownership, permissions and special attributes
date: 2018-10-21
tags: [linux,security]
---

## Ownership

A file or folder has an *owner* and a *group owner*:

```bash
[user:~]$ touch foo
[user:~]$ ls -l foo
-rw-rw-r--. 1 user user Sep 5 18:50 foo
```

In the example above, the "user" (left) is the owner and "user" (right) is the group owner.

### Changing ownership

Ownership can be changed by using the `chown` command:

```bash
[user:~]$ chown henry foo
```

Group ownership can be changed using the `chgrp` command:

```bash
[henry:~]$ chgrp lawyers foo
```

Or by using `chown .<group> <file>`.

To change both ownership and group ownership, use:

```bash
[henry:~]$ chown user.lawyers foo
```

> **Important:** Use the `-R` option to operate on files and directories recursively, making sure ownership is changed for all files and folders on all levels below the parent directory.


## Permissions

Permissions decide who can read, write and execute a file or directory.

```bash
[user:~]$ touch foo
[user:~]$ ls -l foo
-rw-rw-r--. 1 user user Sep 5 18:56 foo
```

The permissions (first) column has the following format:

|- / d|- - -|- - -|- - -| 
|:-----:|:---:|:---:|:---:|
|file/dir| permissions for owner| permissions for group owner |permissions for other users |

In our example, the file can be read and written by owner and group owner, but only read by others.

### chmod (symbolic mode)

`chmod` is used to set set permissions.

```bash
[user:~]$ ls -l foo
-rw-rw-r--. 1 user user Sep 5 18:59 foo
[user:~]$ chmod u+x foo
[user:~]$ ls -l foo
-rwxrw-r--. 1 user user Sep 5 18:59 foo
```

Notice how the owner now has *execute permission* for the file.

Multiple permissions can be specifed in one line:


**User flags**

|flag | meaning |
|:---:|---------|
| u | (user) owner|
| g | group owner|
| o | others|
| a | all the above|
| < blank > | same as `a`, but respect `umask`|

**Operators**

Use `+` to add, `-` to remove and `=` to set explicitly.

**File mode bits**

| bit | effect on files | effect on directories |
|:---:|-------|-------------|
| r   | read  | list files in directory|
| w   | write | create, delete and modify files in the directory|
| x   | execute| go into the directory, access files and subdirs|
| X   | keep `x` if it is already set | set `x`(when bit is used in combination with the `-R` option, subdirectories can be given execute permissions without simultaneously setting execute permissions on files)|
| s  | set user or group ID on execution| only `g+s` (SGID) has effect |
| t   | "sticky bit", ignored by kernel in modern systems | only root, file owner or directory owner can delete and rename files in the directory |


> **Important:** Use the `-R` option to operate on files and directories recursively, making sure permissions are changed for all files and folders on all levels below the parent directory.

### chmod (numeric mode)

```bash
[user:~]$ ls -l foo
-rw-rw-r--. 1 user user Sep 5 18:59 foo
[user:~]$ chmod 700 foo
[user:~]$ ls -l foo
-rwx------. 1 user user Sep 5 18:59 foo
```

As seen above, when `chmod` is supplied with three digits, these represent the permissions of owner, group and others, respectively.
The value of each digit can be 0-7, and the meaning of the value is easily decoded once you understand and remember the following:


|value | permission |
|:----:|------------|
|4 | read |
|2 | write |
|1 | execute |
|0 | none |

Now, the remaining possible values are just combinations of the above:

|value | permission |
|:----:||------------|
|7 | rwx (4+2+1) |
|6 | rw (4+2) |
|5 | rx (4+1) |
|3 | wx (2+1) |


When `chmod` is supplied with four digits, the first digit represent the below (or a combination of these), while the rest of the digits still represent owner, group and others.

|value | permission |
|:----:|------------|
|4 | suid |
|2 | sgid |
|1 | sticky bit |
|0 | none |


### umask

The user file-creation mask (`umask`) is set in `/etc/bashrc` or `/etc/profile`, but users can override default umask in `~/.bashrc` or `~/.bash_profile`.

The current mask can be seen using:

```bash
$ umask
002
$ umask -S
u=rwx,g=rwx,o=rx
```

As seen above, the output is numerical (default) or symbolic (`-S` option).

The value determines the permission that files and directories are assigned when they are created.

Before the value of `umask` is applied, the following is the default:

- 777 is the default permission for dirs
- 666 is the default permission for files

User accounts with UID 200 and above have a default `umask` of 002 (`o-w`, so files will be 664 and dirs 775).
User accounts with UID below 200 have a default `umask` of 022 (`go-w`, so files will be 644 and dirs 755).

```bash
$ umask
002
$ mkdir dir
$ ls -ld dir
drwxrwxr-x. ... # 777 with umask 002 is 775
$ touch foo
$ ls -l foo
-rw-rw-r--. ... # 666 with umask 002 is 664
$ rm -rf dir foo
$ umask 444
$ touch foo
$ ls -l foo
--w--w--w-. ... # 666 with umask 444 is 222
```

## Special file attributes

### Changing special file attributes

Add special file attributes with `+`:

```bash
# chattr +i /etc/fstab
```

Remove special file attributes with `-`:

```bash
# chattr -i /etc/fstab
```

### Listing special file attributes

```bash
$ lsattr
```

Some of the available attributes are:

- a (append only)
- d (no dump - disallow backups of the configured file with the `dump` command)
- e (extent format - set with ext4 filesystem)
- i (immutable)

c, u and s won't work for files on ext4 or XFS.

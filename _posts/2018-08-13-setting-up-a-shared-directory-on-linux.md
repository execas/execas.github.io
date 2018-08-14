---
layout: post
title: "Setting up a shared directory on Linux"
date: 2018-08-12
tags: [linux, configuration, security]
---

This article shows you how to share a directory between users that are members of the same group.

## Create the directory

```bash
# mkdir /home/lawshared
```

## Create the group

### Option 1: Manual

Add line to `/etc/group`:

```
lawyers:x:10001:phil,henry,dana
```

Add line to `/etc/gshadow`:

```
lawyers:!::phil,henry,dana
```

### Option 2: Tool-based

```bash
# groupadd lawyers
```

To add the members to the group, do either:

```bash
# usermod -aG lawyers phil
# usermod -aG lawyers henry
# usermod -aG lawyers dana
```

Or more simply:

```bash
# gpasswd -M phil,henry,dana lawyers
```

## Set ownership and permissions

### Ownership

```
# chown root.lawyers /home/lawshared
```

### Permissions

To set the directory so that the owner (root) and group owner (lawyers) can read, write and execute, but others can do nothing, use:

```bash
# chmod 770 /home/lawshared
```

One thing that is probably not wanted, is that all files created by users are created with the user as owner and group owner, and with standard permissions, so group members can't write to each other's files.

```bash
# su henry
$ cd /home/lawshared
$ touch foo
$ ls -l
-rw-rw-r--. 1 henry henry 0 Aug 12 16:08 foo
```

To solve this, you need to set the SGID bit:

```bash
# chmod g+s /home/testshared
```

Now all files and subdirectories are created with the same group owner as the shared dir, and new directories will inherit the SGID bit:

```bash
# su henry
$ cd /home/lawshared
$ touch foo2
$ mkdir fee
$ ls -l
drwxrwsr-x. 2 henry lawyers 6 Aug 12 16:14 fee 
-rw-rw-r--. 1 henry henry   0 Aug 12 16:08 foo
-rw-rw-r--. 1 henry lawyers 0 Aug 12 16:13 foo2
```

It's possible to set permissions and SGID bit with this one-liner:

```
# chmod 2770 /home/lawshared
```



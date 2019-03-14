---
layout: post
title: Linux file access control lists
date: 2019-03-14
tags: [linux, security]
---

File access control lists (ACLs) go beyond the limitations of regular ugo/rwx permissions by offering more fine-grained access rights, for example setting different permissions for multiple different non-owner users and groups.


## ACL support

A kernel supporting access control lists (>2.6) is necessary:

<div class="term">
~]$ uname -r
</div>

Some filesystems support ACLs, while some don't. Some filesystems need to be mounted with `-o acl`, while others are configured to support ACLs by default (and can be mounted with `-o no_acl` to disable ACL support).

With a newer kernel, xfs, ext4, zfs and tmpfs will likely support ACLs by default. To know for certain, try setting an ACL.

Install the `acl` package, it contains `getfacl` and `setfacl`.

## Setting ACLs

Create a new directory, and remove permissions for everyone except owner (root):

<div class="term">
~]# mkdir /home/shared
~]# chmod 700 /home/shared
</div>

Now, as a regular user, the directory cannot be entered (execute permissions), listed (read permissions) or written to (write permissions).

<div class="term">
~]$ cd /home/shared
bash: cd: shared: Permission denied
~]$ ls /home/shared
ls: cannot open directory /home/shared: Permission denied
~]$ touch /home/shared/foo
touch: cannot touch '/home/shared/foo': Permission denied
</div>

The user "zack" is in the category "others" and therefore has no permissions to the directory. `setfacl` can be used to grant him alone permissions:

<div class="term">
~]# setfacl -m u:zack:x /home/shared
</div>

The `setfacl -m` modifies the ACL for a specified file or directory.
The syntax for modifiying the acl is:

 - `setfacl -m u:<uid/username>:<perms> <file>` for users.
 - `setfacl -m g:<gid/groupname>:<perms> <file>` for groups.

Multiple ACL rules can be set simultaneously by using comma separation:

<div class="term">
~]# setfacl -m u:<user>:rwx,g:<group>:r <file>
</div>

Give "zack" full permissions:

<div class="term">
~]# setfacl -m u:zack:rwx /home/shared
</div>

> Use the `-R` option set the ACL recursively to a directory and its files and subdirectories.

### Removing ACLs

Use `-x` to remove entries from an ACL:

<div class="term>
~]# setfacl -x u:zack:r /home/shared
</div>

## "Getting" ACLs

In a long listing, a plus symbol after the permissions of the file or directory signifies that an ACL is set for that file or directory:

<div class="term">
~]$ ls -ld /home/shared
drwx------+ 2 root root 17 Dec 10 20:15 /home/shared
</div>

`getfacl` can be used to view the ACL of a file or directory:

<div class="term">
~]# getfacl /home/shared
getfacl: Removing leading '/' from absolute path names
# file: home/shared
# owner: root
# group: root
user::rwx
user:zack:rwx
group::---
mask::rwx
other::---
</div>

You can see information that is available through `ls -l` like owner, group and permissions for *user*, *group* and *other*, but also the permissions of specific users and groups (like "zack" having full permissions) and mask.

### Effective rights mask

The effective rights mask, seen above as *mask* in the `getfacl` output, is used to restrict the available operations (read/write/execute) of user, group, others, specific users and specific groups. The mask denotes the maximum access rights. The mask entry was automatically added when `setfacl` was used to add "zack" to the ACL.

Since "zack" has full permissions, and the mask is rwx, he can for example do the following:

</div>
~]$ touch /home/shared/test1
</div>

The mask can be modified using `setfacl`:

<div class="term">
~]# setfacl -m m:rx /home/shared
</div>

Now, since the mask (think *maximum access rights*) is `rx`, "zack" can't write to the shared directory:

<div class="term">
~]$ touch /home/shared/test2
touch: cannot touch '/home/shared/test2': Permission denied
</div>

Notice that the mask value is changed:

<div class="term">
~]# getfacl /home/shared/
getfacl: Removing leading '/' from absolute path names
# file: home/shared/
# owner: root
# group: users
user::rwx
user:zack:rwx                   #effective:r-x
group::r-x
mask::r-x
other::---
</div>

> The mask is recalculated every time an ACL entry is added or removed to "include the union of all permissions affected by the mask entry". Use the `-n` option of `getfacl` if this is not wanted.

> Be aware that `ls -l` does not display the effective rights (a mask in the ACL may restrict effective rights).

## Default ACLs

Default ACLs can only be set on directories. A directory will inherit its parent's default ACL as its ACL and default ACL. A file will inherit the parent's default ACL as its ACL.

<div class="term>
~]# setfacl -d -m u:zack:r /home/shared
</div>

This will:

    - add a default ACL for "zack" as specified
    - add default ACLs for user, group, other and mask
    - NOT change the existing entries in the ACL
    - NOT change the ACL for existing files and dirs in /home/shared

Use `setfacl -k <dir>` to remove default ACLs.

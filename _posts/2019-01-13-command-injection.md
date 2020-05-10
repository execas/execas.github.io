---
layout: post
title: "Command injection"
date: 2019-01-13
tags: [security, linux, programming, vulnerabilities]
---

Programs vulnerable to command injection attacks allow an attacker to execute arbitrary commands, possibly remotely and with high privileges.

## Classic example

Below is a classic example, written in C.

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[])
{
    if (argc < 2)
        return -1;

    char cmd[100];
    snprintf(cmd, 100, "cat %s | sed 's/@/ at /'", argv[1]);
    system(cmd);
}
```

> Here, `system()` is used, but there are a bunch of functions in C and other languages that can execute a file (`execl()`, `popen()`, `os.popen()` and so on). There are also functions which interpret data as code (like Python's `eval()`).

### Expected result

The program outputs a user specified file, substituting "@" with " at ".

```bash
$ ./mailtool accounts
test at example.com
user at example.org
john at example.org
judy at example.org
```

### Exploitation

An attacker could easily inject any command for execution by the program:

```
$ ./mailtool "accounts;cat /etc/passwd"
test@example.com
user@example.org
john@example.org
judy@example.org
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
...
```

The command injection could be anything, like a bind or reverse shell with `nc`, deletion of important files, output of information and so on, depending on the privileges the program runs with. A suid root program is of course the worst possible scenario, as there is no limit to what an attacker is able to do.

The vulnerable programs can be anything from small utility programs and tools, to a web application which executes system commands based on user input in its back end.

## Mitigation

The problem boils down to input validation and sanitization.

If possible, do not use user input as part of commands. If not, take note of the following:

### Whitelist

Decide what sort of data should be accepted.
Reject or transform data that does not match whitelisting pattterns.

For the program above, we could for example accept only lowercase characters, or we could also include digits and some special characters like **/**, **.**, **~** and so on.

The risk is accepting to much (e.g. some special characters, like **;** in this case) or not enough (e.g. **_** and **-** may be needed).

> Regular expressions (regex) is one way of defining whitelisting patterns.

> Rejecting is in most cases a better choice than transforming.

### Blacklist

An alternative to whitelisting, and often considered inferior.

Decide what sort of data should be rejected.
Accept all data not matching blacklisting patterns (and perhaps transform other data until they match).

For the program above, we could for example reject **;**, **&** and the space character.

The risk is rejecting to little (can you think of all possibilities hackers could discover?) or too much.


### Validate

Can you validate the input somehow? For the example program above we could check if the specified file exists first, but this may also suffer from the same vulnerability if not done carefully.

### Identify all inputs

It's important to identify all inputs to be able to validate and sanitize all user-supplied data.

If the program above implemented whitelisting to only accept safe inputs, an attacker could still exploit it by taking advantage of the program not specifying the full path of `cat`:

```c
$ echo "echo Hijacked!" > cat
$ chmod +x cat
$ PATH=.:$PATH
$ ./mailtool accounts
Hijacked!
```

This could be mitigated by instead using the full path, e.g. `/usr/bin/path`.

Programs may use data from for example environment variables and files that can be modified by a user, so these can also be vectors in a command injection attack.

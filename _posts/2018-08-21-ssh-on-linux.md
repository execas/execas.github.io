---
layout: post
title: "Using SSH on Linux"
date: 2018-08-21
tags: [linux, security]
---

## Using SSH

### Connecting to a remote system

Connect to a remote system as current user:

```bash
[user@example.com:~]$ ssh example.net
user@example.net's password:
```

Connect to a remote system as a specified user:

```bash
[user@example.com:~]$ ssh henry@example.net
henry@example.net's password:
```

### Copying files to and from remote systems

Copy a file to a the specified user's home dir on a remote system:

```bash
[user@example.com:~]$ scp hello.txt henry@example.net:~
henry@example.net's password:
```

If no path is specified, home dir is default:

```bash
[user@example.com:~]$ scp hello.txt henry@example.net:
henry@example.net's password:
```

Copy a file from a remote system to the current system:

```bash
[user@example.com:~]$ scp henry@example.net:hello.txt .
henry@example.net's password:
```

Copy a file between remote systems:

```bash
[user@example.com:~]$ scp henry@example.net:hello.txt phil@example.org:
henry@example.net's password:
phil@example.org's password:
```

Use `-r` to copy directories with files and subdirectories:

```bash
[user@example.com:~]$ scp -r henry@example.net:myfolder .
henry@example.net's password:
```

## SSH configuration

The configuration files for ssh is `/etc/ssh/sshd_config` and `/etc/ssh/ssh_config`.

### Root login

Uncomment the below in `sshd_config` to allow for root login over ssh:

```
#PermitRootLogin yes
```

Restart the ssh service.

A much better choice is to not allow root to login over ssh, and instead only allow regular users to gain extended priveleges through `sudo`.

### Passwords

Passwords are easy to guess and easy to steal. A better option is to use SSH keys for authentication.

1) Create SSH key on your machine:

```bash
$ ssh-keygen -a 100 -t ed25519
```

Default: The private key is `id_ed25519` and the public key is `id_ed25519.pub`, both located in `~/.ssh/`.  Use `-f <name>` option to create and name keys for different purposes.

Enter a passphrase to protect your private key, in case it ends up in the wrong hands.

2) Copy SSH key to other machines:

```bash
[user@example.com:~]$ $ ssh-copy-id user@example.org
```

Use `-i <name>.pub` to copy a specific public key.

3) Disallow password logins (Optional):

Change `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
```

Restart the ssh service.







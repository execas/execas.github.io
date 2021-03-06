---
layout: post
title: Reset root password on Linux
date: 2018-09-03
tags: [linux,security]
---

The following shows how easy it is to reset the root password on a Linux system. Disabling booting from external devices and setting a boot password can hinder someone using this method with malicious intent.

## Booting into emergency mode

**Emergency mode** gives you root privileges without a root password. We will enter this mode to reset the root password.

### Reboot

Reboot the sytem, and press `e` at the GRUB boot loader screen.

### Edit parameters

Find the line beginning with `linux16`, and append `rd.break` to the end of that line.

Press `Ctrl-x` to boot into emegency mode.

## Resetting the root password

### Remount sysroot

```bash
# mount -o remount,rw /sysroot
```

### chroot jail

```bash
# chroot /sysroot
```

### Set a new password

```bash
# passwd
```

### SELinux relabel

```bash
# touch /.autorelabel
```

## Reboot

Press `Ctrl-d` to leave the `chroot jail`. Type `reboot` and press enter.
The new password for `root` can now be used.

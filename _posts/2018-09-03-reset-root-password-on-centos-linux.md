---
layout: post
title: Reset root password on CentOS Linux
date: 2018-09-03
tags: [linux,security]
---

## Booting into emergency mode

Emergency mode gives you root priveleges without a root password. We will use this mode to create reset the root password.

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

## SELinux relabel

```bash
# touch /.autorelabel
```

## Reboot

Press `Ctrl-d` to leave the `chroot jail`. Type `reboot` and press enter.

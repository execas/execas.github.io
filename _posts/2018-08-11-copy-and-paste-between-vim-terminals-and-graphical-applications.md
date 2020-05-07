---
layout: post
title: Copy and paste between vim, terminals and graphical applications
date: 2018-08-11
tags: [vim, linux, tools]
---

Being able to move text between web browsers, terminals and vim is essential for programmers.

## vim clipboard support

On some distributions, `vim` is not compiled with clipboard support. Use the command below to check your vim.

```bash
$ vim --version | grep clipboard
```

If **clipboard** and **xterm-clipboard** has a plus symbol in front of them, vim is compiled with clipboard support and you can skip ahead to the next part.

### No clipboard support: Alternative 1

*gvim* is compiled with clipboard support.
If your system comes with gvim, you can use `gvim -v` as a substitute for `vim`. Put the following in your `.bashrc`:

```
alias vim='gvim -v'
```

### No clipboard support: Alternative 2

Get a copy of *vim* that's been compiled with clipboard support, or compile it yourself.

To compile vim with clipboard support, you need the sources for vim as well as `dbus-x11` and `libx11-devel`.

If you for some reason choose to use an older source, make sure you include the following parameter during configuration:

```bash
$ ./configure --with-features=huge
```

## About copying and pasting in vim

vim uses **y**, **d** and **p**, combined with movement or selection commands, to copy, cut and paste.

### Registers

You can copy or cut to, or paste from, a specified register, e.g. `"adw` cuts to register *a*, `"ap` pastes from register *a*. Lower case letter registers are most commonly used.

Use `:reg` to view the registers.
Some registers serve as a history, and some are special purpose. Macros are also stored in user named registers.

### Clipboard registers

When vim is compiled with clipboard support, the **\*** and **+** registers are available, and can be accessed using `"*` and `"+`.
Terminal applications typically use the *\** register for copying and pasting, and graphical applications, like Firefox, typically use the *+* register.


## Configuration for easy copy/paste

To easily copy text to and from vim and other applications, put the following in your `.vimrc`:

```
set clipboard=unnamed,unnamedplus
```
The above line gives you the following behavior:

  - every time you cut or copy in vim (without specifying a register) this text is copied to the *\** and *+* registers, and can be pasted into a terminal application (using `shift-insert`) or graphical applications (using `C-v` or `shift-insert`).
  - copying from graphical applications will let you paste into vim without specifying a register.
 
Copying from terminal applications (highlighting the text and using `C-M-c`) typically fills the *\** register, but most terminals can be configured to use *+*:


### xterm clipboard support

To enable `xterm` to use the clipboard (*+* register in vim), put the following in your `.Xresources`:

```
xterm*selectToClipboard: true
```

### urxvt clipboard support

To enable `urxvt` to use the clipboard (*+* register in vim), put the following in your `.Xresources`:

```
URxvt.perl-ext-common: selection-to-clipboard
```

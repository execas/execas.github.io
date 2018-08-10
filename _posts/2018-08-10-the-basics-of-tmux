---
layout: post
title: "The basics of tmux"
date: 2018-08-10
tags: [linux, tools, tmux]
---

**tmux** is a terminal multiplexer. It makes using the terminal more user friendly, more flexible and more fun.

## Sessions

Sessions are used to create separate workspaces for different activities.

### Starting a new session

To start a new *tmux* session, execute the following command in the terminal:

```
$ tmux new -s <sessionName>
```

Alternatively you can skip naming the session and it will be assigned a number instead of a name.

```
$ tmux
```

### Renaming a session

While in a session, you can rename it by using `C-b $ <newSessionName>`. Alternatively you can rename a session using:

```
$ tmux rename-session -t <oldName> <newName>
```

### Detaching from a session

Use `C-b d` to detach from a session.

### Listing sessions

While in a session, use `C-b s` to open the session selector. Alternatively you can list sessions using:

```
$ tmux list-sessions
```

Or use the shorthand:

```
$ tmux ls
```

### Attaching to a session

From the command line you can use:

```
$ tmux attach -t <sessionName>
```

Or use the shorthand:

```
$ tmux a -t <sessionName>
```

You can attach to the previously detached session using:

```
$ tmux a
```

While in a session, use the `C-b s` session selector.

### Killing a session

If the session has no windows left, it will be killed automatically. Alternatively you can use:

```
$ tmux kill-session -t <sessionName>
```

## Windows

Sessions can contain multiple windows. If you just created a session, it will have one window. Each new window has a fresh prompt.
By default a list of the session's windows is visible on the last line, with a star marking the current window.

### Creating a new window

While in a session, use `C-b c` to create a new window.

### Renaming windows

To rename the current window, use `C-b , <newWindowName>`. Alternatively you can rename a window using:

```
$ tmux rename-window -t <oldName> <newName>
```

### Switching between windows

To cycle through windows, use `C-b n` and `C-b p`.

A window selector can be launched by using `C-b w`.

Use `C-b <n>` to switch to window number \<n\>.

Use `C-b f <string>` to find a window containing \<string\>.

### Killing a window

Exit the prompt using `C-d`, or use `C-b &`.

## Panes

Each split is called a pane, and contains a fresh prompt.

### Splitting a window into panes

To split horizontally, use `C-b "`.

To split vertically, use `C-b %`.

### Switching between panes

Use `C-b o` to cycle between the panes.

Use `C-b ;` to go to last pane.

### Moving panes

Use `C-b C-o` to rotate the split pane.

Use `C-b {` and `C-b }` to swap panes.

### Layouts

To maximize the current pane, use `C-b z`. Repeat to restore layout.

Use `C-b space` to cycle between useful layouts.

Use `C-b C-<arrow>` to resize panes.

### Killing a pane

Use `C-b x` to kill a pane and its window.

## Help

While in a session, use `C-b ?` to view all shortcuts. Tmux also has a great man-page.

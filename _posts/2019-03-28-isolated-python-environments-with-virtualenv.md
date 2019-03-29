---
layout: post
title: Isolated Python environments with virtualenv
date: 2019-03-28
tags: [python, programming]
---

Isolated Python environments are very useful for making sure that a project's dependencies and version requirements are fullfilled without affecting other projects or the host itself. It also makes it possible to install needed Python packages on a host without full admin access.

## Installation

If `virtualenv` is not installed, get it using `pip`:

<div class="term">
<b>~]# pip install virtualenv</b>
</div>

> If `pip` is not installed, do `curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py` and then `python get-pip.py`, or try installing it using your package manager.


## Usage

To create an isolated ('virtual') environment, do the following:

<div class="term">
<b>~]$ virtualenv example</b>
New python executable in example
Installing setuptools, pip, wheel...done.
</div>

In addition to the files already in the directory, we now got the following:

- bin/
- include/
- lib/
- lib64/

The above command has prepared our isolated environment, but it isn't *active* yet.

### Activating the isolated environment

Step inside the project directory:

<div class="term">
<b>~]$ cd example</b>
</div>

On this system, we are using the default python installation, and `pip` isn't installed:

<div class="term">
<b>example]$ which python</b>
/bin/python
<b>example]$ which pip</b>
/usr/bin/which: no pip in (/usr/bin:/usr/sbin...
</div>

When we activate the enviroment, things change:

<div class="term">
<b>example]$</b> source bin/activate
<b>(example) example]$</b> which python
~/example/bin/python
<b>(example) example]$</b> which pip
~/example/bin/pip
</div>

Now, we are free to install (specific versions of) packages using `pip install`, without worrying about how this will affect other projects or the system.

Notice that the project directory name appears inside parentheses as a reminder that we have activated an isolated environment for the current terminal.

### Deactiving the isolated environment

When we're finished working on the project, we put things back to normal by deactivating the isolated environment:

<div class="term">
<b>(example) example]$</b> deactivate
<b>example]$</b> which python
/bin/python
</div>

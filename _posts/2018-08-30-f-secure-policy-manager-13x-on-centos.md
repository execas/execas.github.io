---
layout: post
title: "F-Secure Policy Manager 13.x on CentOS"
date: 2018-08-30
tags: [linux, security]
---

*Policy Manager* is the tool used to manage and monitor hosts running F-Secure security software.

## Getting the software

Download RPM's for **Policy Manager Server** and **Policy Manager Console** here:

<https://www.f-secure.com/en/web/business_global/downloads/policy-manager-for-linux>

>**Note:** The software requires and includes Java™ Runtime Environment (JRE). 


### F-Secure Policy Manager Server

The server stores policies, status information and more, and communicates configuration changes and status with hosts through `https`and `http` (non-sensitive data).

### F-Secure Policy Manager Console

The administration console is a GUI application used to configure, monitor and install F-Secure on hosts.
The console can be installed on multiple machines, Windows or Linux.


## Installing prerequisites

### `net-tools`

```bash
# yum install net-tools
```

### `libstdc++.i686`

```bash
# yum install libstdc++.i686
```

## Installing F-Secure Policy Manager

### Server

**1. Install:**

```bash
# rpm -iv fspms*
```

**2. Configure:**

```bash
# /opt/f-secure/fspms/bin/fspms-config
```

**3. Enable:**

```bash
# systemctl enable fspms
```

### Console

**1. Install:**

```bash
# rpm -iv fspmc*
```

**2. Add users to group:**

```bash
# gpasswd -M phil,henry,dana fspmc
```

## Using Policy Manager

### Launch console

```bash
$ /opt/f-secure/fspmc/fspmc
```

The default username is `admin`.

### Admin guide

<https://help.f-secure.com/product.html#business/policy-manager/13.10/en/concept_5C26F4BF4D3149F5A17056AAE157EB68-13.10-en>









---
layout: post
title: HTTP in Linux
date: 2018-12-03
tags: [linux,networking, security]
---


HTTP (Hypertext Transfer Protocol) is a protocol used to exchange data between a client and a web server.

### Install and activate the Apache web server

```bash
~]# yum install httpd
~]# systemctl start httpd
~]# systemctl enable httpd
```

The configuration file is `/etc/httpd/conf/httpd.conf`.


### SELinux

The directories `/var/www/cgi-bin` and `/var/www/html` are created during the installation with the following SELinux contexts:

```bash
~]# ls -Z /var/www
drwxr-xr-x. root root system_u:object_r:httpd_sys_script_exec_t:s0 cgi-bin
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 html
```

Html-files can be copied to the `/var/www/html` directory to be served by the httpd daemon.

Files labeled with `httpd_sys_script_exec_t` can be executed by `httpd`, and files labeled with `httpd_sys_content` can be read by `httpd` and the scripts it executes (permissions and attributes must also be correctly set).

Use `semanage fcontext -l | grep /var/www` see all file contexts.

> If `semanage` is not found, try installing `policycoreutils-python`.


### Grant access from remote systems

```bash
~]# firewall-cmd --permanent --add-service=http
~]# firewall-cmd --reload
```

### Clients

Use `wget`, `curl` or a web browser.

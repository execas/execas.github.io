---
layout: post
title: FTP in Linux
date: 2018-12-02
tags: [linux,networking, security]
---

## FTP

FTP (file transfer protocol) is a protocol used to transfer files between a client and a server.

### Install and activate a server

```bash
# yum install vsftpd
# systemctl start vsftpd
# systemctl enable vsftpd
```

The configuration file is `/etc/vsftpd/vsftpd.conf`.


### SELinux

The data directory `/var/ftp/pub` is created during installation with the following SELinux context:

```
# ls -Z /var/ftp
drwxr-xr-x. root root system_u:object_r:public_content_t:s0 pub
```

The `public_content_t` label makes the files and foldes copied to `/var/ftp/pub` readable by clients (file permissions and attributes must also be correctly set).

All files and folders copied to `/var/ftp/pub` get the following type context labels:

```
# semanage fcontext -l | grep /var/ftp
/var/ftp(/.*)?                                     all files          system_u:object_r:public_content_t:s0 
/var/ftp/bin(/.*)?                                 all files          system_u:object_r:bin_t:s0 
/var/ftp/etc(/.*)?                                 all files          system_u:object_r:etc_t:s0 
/var/ftp/lib(/.*)?                                 all files          system_u:object_r:lib_t:s0 
/var/ftp/lib/ld[^/]*\.so(\.[^/]*)*                 regular file       system_u:object_r:ld_so_t:s0 
```

To make ftp clients able to write to a directory, change the type context to `public_content_rw_t`. Anonymous writes need the boolean `allow_ftpd_anon_write`.

> If `semanage` is not found, try installing `policycoreutils-python`.


### Grant access from remote systems

```bash
# firewall-cmd --permanent --add-service=ftp
# firewall-cmd --reload
```

### Clients

Use `lftp` (fast), `sftp` (secure) or a web browser.

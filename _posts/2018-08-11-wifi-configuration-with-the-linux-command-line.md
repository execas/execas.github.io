---
layout: post
title: "Wifi-configuration with the Linux command line"
date: 2018-08-11
tags: [linux,configuration]
---

Most of the commands below will require super user privileges.

### Get a list of wireless interfaces

```bash
# iw dev
```

### Enable a wireless interface

Check the state of the interface:

```bash
# ip link show dev <wireless interface>

Bring the interface up:

``` bash
# ip link set dev <wireless interface> up
```

### Find available wireless networks

``` bash
# iw <wireless interface> scan | grep SSID
```

### Configure authentication

Generate and store the authentication configuration for the SSID you want to connect to (if this hasn't been done before). 
Run the command below, enter the passphrase and hit return.

``` bash
# wpa_passphrase <SSID> >> /etc/wpa_supplicant.conf
```

### Authentication

``` bash
# wpa_supplicant -B -D wext -i <wireless interface> -c /etc/wpa_supplicant.conf
```

### Get an IP address

``` bash
# dhclient <wireless interface>
```

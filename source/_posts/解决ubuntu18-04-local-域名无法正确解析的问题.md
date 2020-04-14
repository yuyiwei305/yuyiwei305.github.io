---
title: 解决ubuntu18.04 .local 域名无法正确解析的问题
date: 2020-04-14 21:44:06
tags: [ubuntu,dns]
---


### 问题描述

今天发现有一台机器dns解析总是出现问题，经过排查发现ubuntu18.04更改了域名解析的规则，默认是走127.0.0.53 这个地址，由它再去到真正的dns地址。同时系统配置了一些保留域名，导致系统拒绝为一些域名提供解析。报错如下：

```
root@ip-xx-xx-xx-xx:~# dig xxx.local

; <<>> DiG 9.11.3-1ubuntu1.11-Ubuntu <<>> xxx.local
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 49542
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
; xxx.local.	IN	A

;; Query time: 0 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Tue Apr 14 13:26:14 UTC 2020
;; MSG SIZE  rcvd: 56
```
<!-- more -->

同时我们可以通过命令查询到：

```
root@ip-xx-xx-xx-xx:~# systemd-resolve --status
Global
          DNSSEC NTA: 10.in-addr.arpa
                      16.172.in-addr.arpa
                      168.192.in-addr.arpa
                      17.172.in-addr.arpa
                      18.172.in-addr.arpa
                      19.172.in-addr.arpa
                      20.172.in-addr.arpa
                      21.172.in-addr.arpa
                      22.172.in-addr.arpa
                      23.172.in-addr.arpa
                      24.172.in-addr.arpa
                      25.172.in-addr.arpa
                      26.172.in-addr.arpa
                      27.172.in-addr.arpa
                      28.172.in-addr.arpa
                      29.172.in-addr.arpa
                      30.172.in-addr.arpa
                      31.172.in-addr.arpa
                      corp
                      d.f.ip6.arpa
                      home
                      internal
                      intranet
                      lan
                      local
                      private
                      test

Link 2 (ens5)
      Current Scopes: DNS
       LLMNR setting: yes
MulticastDNS setting: no
      DNSSEC setting: no
    DNSSEC supported: no
         DNS Servers: xx.xx.xx.xx
          DNS Domain: xx.compute.internal
```
### 问题处理

可以看出 .local 是被保留成本地的了。 知道这些我们开始处理。

1. 首先修改默认的dns为自己设置，修改/etc/systemd/resolved.conf DNS字段为我们要的dns地址 这里以8.8.8.8为例：

```
# cat /etc/systemd/resolved.conf

#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.
#
# Entries in this file show the compile time defaults.
# You can change settings by editing this file.
# Defaults can be restored by simply deleting this file.
#
# See resolved.conf(5) for details

[Resolve]
DNS=8.8.8.8
#FallbackDNS=
#Domains=
#LLMNR=no
#MulticastDNS=no
#DNSSEC=no
#Cache=yes
#DNSStubListener=yes

```

2. 重启 systemd-resolved

```
# systemctl restart systemd-resolved.service
```

3. 配置软连接使其生效

```
ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```
4. 检查看是否生效,不多说直接dig 或者 nslookup看解析是否正确即可

```
dig xxx.local
```

到此这个问题处理完成。

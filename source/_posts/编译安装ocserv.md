---
title: 编译安装ocserv
date: 2020-06-18 20:33:06
tags: [ocserv,vpn,newwork]
---
### 简单介绍
  ocserv 是目前比较常用的vpn软件。这里完整记录下从零开始搭建过程。

  名称|值|说明
  -|-|-|
  操作系统 | ubuntu 16.04  |  所用到的基础image |

其实没啥需求，直接装就行。这里不废话直接给出命令，搞成脚本一件执行即可。

### 安装ocsserv
```
#依赖的安装
apt-get update
apt-get -y  install wget git dh-autoreconf  libgnutls28-dev libev-dev libwrap0-dev  libpam0g-dev  liblz4-dev libseccomp-dev   libreadline-dev libnl-route-3-dev libnl-route-3-dev libkrb5-dev  libradcli-dev libprotobuf-c-dev libtalloc-dev libhttp-parser-dev libpcl1-dev protobuf-c-compiler gperf liblockfile-bin nuttcp lcov libuid-wrapper libpam-wrapper  libnss-wrapper libsocket-wrapper gss-ntlmssp haproxy iputils-ping freeradius gawk yajl-tools

#下载源代码
git clone https://gitlab.com/openconnect/ocserv.git

#编译安装
cd ocserv && autoreconf -fvi && ./configure && make
cp src/ocserv /usr/sbin/ocserv
cp src/ocserv-worker  /usr/sbin/ocserv-worker
cp src/ocserv-fw /usr/bin/ocserv-fw
cp src/occtl/occtl /usr/bin/
cp src/ocpasswd/ocpasswd /usr/bin/
cp doc/systemd/standalone/ocserv.service  /etc/systemd/system/ocserv.service

#内核参数调优
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.core.default_qdisc = fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control = bbr" >> /etc/sysctl.conf
sysctl -p

```
<!-- more -->

### 安装证书

上述步骤执行完基本的ocserv 已经安装好， 其实如果不编译 直接执行 apt 安装也行

```
apt-get update && apt-get -y install  ocserv
```

不知道为啥，上面编译执行完之后有的时候会出现dns解析有问题，尤其是高版本ubuntu。所以在执行下面的步骤前，最好检查下dns，或者干脆重启

之后安装证书 两种方式 二选一,哪种方式好使用哪种，总之要把certbot装上

```
# 1. 第一种
apt-get update
apt-get  -y install software-properties-common
add-apt-repository ppa:certbot/certbot -y
apt-get update
apt-get -y install certbot

# 2.第二种
apt-get update
apt-get -y install certbot

```
申请证书
```
# 域名是你自己的，提前做好解析
export domain = xxx.com
certbot certonly --standalone -n --agree-tos --email xxx@xxx.com  --preferred-challenges http -d $domain
```
证书申请成功长这样：

```
- Congratulations! Your certificate and chain have been saved at:
  /etc/letsencrypt/live/xxx.com/fullchain.pem (1)
  Your key file has been saved at:
  /etc/letsencrypt/live/xxx.com/privkey.pem (2)
  Your cert will expire on 2020-09-14. To obtain a new or tweaked
  version of this certificate in the future, simply run certbot
  again. To non-interactively renew *all* of your certificates, run
  "certbot renew"
- Your account credentials have been saved in your Certbot
  configuration directory at /etc/letsencrypt. You should make a
  secure backup of this folder now. This configuration directory will
  also contain certificates and private keys obtained by Certbot so
  making regular backups of this folder is ideal.
- If you like Certbot, please consider supporting our work by:


(1) (2) 这里是我们一会儿要用到的
```

### 配置服务

1. 修改配置文件
```
# 下载个网上现成的，具体优化参考官方的 sample 也行，这里就不优化了
wget -N --no-check-certificate -P "/etc/ocserv" "https://files.zorz.cc/ocserv.conf"


auth = "plain[passwd=/etc/ocserv/ocpasswd]"
# listen-host = [IP|HOSTNAME]
tcp-port = 443
#udp-port = 443  #最好注释掉（0）
run-as-user = nobody
run-as-group = daemon
socket-file = /var/run/ocserv-socket
server-cert = /etc/letsencrypt/live/xxx.com/fullchain.pem （1）
server-key = /etc/letsencrypt/live/xxx.com/privkey.pem   （2） #刚才写的
ca-cert = /etc/ocserv/ssl/ca-cert.pem
isolate-workers = true
banner = "Hello"
max-clients = 16
max-same-clients = 2
server-stats-reset-time = 604800
keepalive = 32400
dpd = 30
mobile-dpd = 90
switch-to-tcp-timeout = 25
try-mtu-discovery = true
tls-priorities = "NORMAL:%SERVER_PRECEDENCE:%COMPAT:-VERS-SSL3.0"
auth-timeout = 240
min-reauth-time = 300
max-ban-score = 80
ban-reset-time = 1200
cookie-timeout = 300
deny-roaming = false
rekey-time = 172800
rekey-method = ssl
use-occtl = true
pid-file = /var/run/ocserv.pid
device = vpns
predictable-ips = true
default-domain = xxx.com #(3)

ipv4-network = 192.168.169.0 #(4)
ipv4-netmask = 255.255.255.0
# An alternative way of specifying the network:
#ipv4-network = 192.168.1.0/24
# The IPv6 subnet that leases will be given from.
#ipv6-network = fda9:4efe:7e3b:03ea::/48
# Specify the size of the network to provide to clients. It is
# generally recommended to provide clients with a /64 network in
# IPv6, but any subnet may be specified. To provide clients only
# with a single IP use the prefix 128.
#ipv6-subnet-prefix = 128
#ipv6-subnet-prefix = 64
tunnel-all-dns = true
dns = 8.8.8.8
dns = 8.8.4.4
ping-leases = false
# route = 10.10.10.0/255.255.255.0
# route = 192.168.0.0/255.255.0.0
# route = fef4:db8:1000:1001::/64
# route = default
# no-route = 192.168.5.0/255.255.255.0
cisco-client-compat = true
dtls-legacy = true



# 需要改的我已经标注好了
```
2. 添加用户以及启动

```
iptables -t nat -A POSTROUTING -j MASQUERADE

systemctl enable ocserv

ocpasswd -c /etc/ocserv/ocpasswd test

systemctl start ocserv

```
3. 查看日志

```
journalctl -xe -u ocserv.service
```

至此服务基本搭建完成

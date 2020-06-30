---
title: dnsmasq ipset iptables 实现对流量进行分流
date: 2020-06-29 20:45:48
tags: [ocserv,策略路由,vpn]
---

### 需求和实现方式


  使用vpn相比代理方式的确会稳定，但是常常会导致客户端网络改变而造成一些问题，所以我们更希望同时具有vpn的稳定以及基于不同网络访问请求而走不同流量的策略。我们可以通           
过 dnsmasq+ipset区分出不同域名的不同ip，并且对ip进行分组，然后使用iptable 对不同组的ip打上不同的标签，最后再走不同的路由表实现策略路由。 基本的流程图如下:


```mermaid
graph LR
    正常流量 --> dnsmasq --> ipset --> iptables --> iproute --> 流量转发  
```


### 配置流程
  1. 创建 ipset

  这里创建两个：
  ```
  #安装
  apt-get update && apt-get install -y ipset

  #配置（使用这个即可 下面的都是一些常用命令）
  ipset create gfwlist2 hash:ip
  ipset create gfwlist3 hash:ip

  #可以通过以下命令查看
  ipset list

  # 保存
  ipset save | tee /etc/ipset.conf
  ```
<!-- more -->
  2. DNSMASQ 配置
  主要就是两个选项 server 和 ipset ，server决定域名走什么dns，ipset指定查询的结果放入的ipset。
  这里给出例子：
  ```
  server=/cn/114.114.114.114#53    
  ipset=/cn/gfwlist2       
  server=/.google.com/8.8.8.8#53
  ipset=/.google.com/gfwlist3
  ```

  3. 路由表，IPTABLES, IP RULE

  创建路由表:
  ```
  echo "55 gfwlist2" | tee -a /etc/iproute2/rt_tables
  echo "66 gfwlist3" | tee -a /etc/iproute2/rt_tables
  ```

  iptables 规则:
  ```
  iptables -t mangle -N fwmark2
  iptables -t mangle -N fwmark3

  iptables -t mangle -I PREROUTING -m set --match-set gfwlist2 dst -j MARK --set-mark 2
  iptables -t mangle -I PREROUTING -m set --match-set gfwlist3 dst -j MARK --set-mark 3

  iptables -t mangle -A OUTPUT -j fwmark2
  iptables -t mangle -A OUTPUT -j fwmark3

  iptables -t mangle -A fwmark2 -m set --match-set gfwlist2  dst -j MARK --set-mark 2
  iptables -t mangle -A fwmark3 -m set --match-set gfwlist3  dst -j MARK --set-mark 3


  iptables -t nat -A POSTROUTING -m mark --mark 0x2 -j MASQUERADE
  iptables -t nat -A POSTROUTING -m mark --mark 0x3 -j MASQUERADE

  ```

  具体的解释留个坑 回头填...

  ip route:
  ```
  #添加路由
  ip route add default  via 192.168.171.1 dev tun1 table 55  
  ip route add default  via 192.168.169.1 dev tun1 table 66
  ```

  ip RULE:
  ```
  ip rule add fwmark 2 table gfwlist2 #标记2的流量走路由表 55（gfwlist2）
  ip rule add fwmark 3 table gfwlist3 #标记3的流量走路由表 66（gfwlist3）
  ```

  至此 配置全部结束。 策略路由已实现。 可以试试啦～

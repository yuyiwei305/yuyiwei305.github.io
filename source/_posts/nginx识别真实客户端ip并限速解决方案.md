---
title: nginx识别真实客户端ip并限速解决方案
date: 2020-04-22 14:14:48
tags: [nginx]
---
最近公司某些接口经常遭到cc攻击，暂时可以通过配置nginx来缓解这个问题，在这里记录下处理的思路和流程。

整个环节分为两个步骤，首先识别出真正的客户端ip地址，然后针对这些地址去做访问速率限制，禁止频繁请求，在这里我们使用ngx_http_limit_req_module和ngx_http_limit_conn_module 来限制资源请求。

###  识别客户端地址

由于大量使用了cdn以及经过了层层代理，直接使用nginx的$remote_addr无法得到真正的客户端ip，显示的大概率是上层代理的ip。所以在这里我们使用了http_realip_module这个模块来解决这个问题。

http_realip_module 由三个部分组成：

1、set_real_ip_from 是指接受从哪个信任前代理处获得真实用户ip

2、real_ip_header 是指从接收到报文的哪个http首部去获取前代理传送的用户ip

3、real_ip_recursive 是否递归地排除直至得到用户ip（默认为off） 

首先，real_ip_header 指定一个http首部名称，默认是X-Real-Ip，假设用默认值的话，nginx在接收到报文后，会查看http首部X-Real-Ip。

（1）如果有1个IP，它会去核对，发送方的ip是否在set_real_ip_from指定的信任ip列表中。如果是被信任的，它会去认为这个X-Real-Ip中的IP值是前代理告诉自己的，用户的真实IP值，于是，它会将该值赋值给自身的$remote_addr变量；如果不被信任，那么将不作处理，那么$remote_addr还是发送方的ip地址。

（2）如果X-Real-Ip有多个IP值，比如前一方代理是这么设置的：proxy_set_header X-Real-Ip $proxy_add_x_forwarded_for;

得到的是一串IP，那么此时real_ip_recursive 的值就至关重要了。nginx将会从ip列表的右到左，去比较set_real_ip_from 的信任列表中的ip。如果real_ip_recursive为off，那么，当最右边一个IP，发现是信任IP，即认为下一个IP（右边第二个）就是用户的真正IP；如果real_ip_recursive为on，那么将从右到左依次比较，知道找到一个不是信任IP为止。然后同样把IP值复制给$remote_addr。

了解了这个原理接下来就比较好操作了。
<!-- more -->
第一步，设置 set_real_ip_from  ： 由于我们采用的是aws的cdn服务，直接查询AWS 的 IP 地址范围，官方是提供了一个地址可以下载到的 https://docs.aws.amazon.com/zh_cn/general/latest/gr/aws-ip-ranges.html ，将其修改成 set_real_ip_from
这个格式的文件。 我这里将其命名为 realip.txt ,这里给出一些例子。 其中内网ip 是我自己加的，其他的都是官方给的。
```
set_real_ip_from  127.0.0.1;
set_real_ip_from  10.0.0.0/8;
set_real_ip_from  13.248.118.0/24;
set_real_ip_from  18.208.0.0/13;
set_real_ip_from  52.95.245.0/24;
set_real_ip_from  54.240.17.0/24;
set_real_ip_from  99.77.142.0/24;
set_real_ip_from  99.77.187.0/24;
set_real_ip_from  52.194.0.0/15;
...

```
第二步： 设置 real_ip_header 为 X-Forwarded-For，这个没什么好说的X-Forwarded-For基本上已经成了事实标准。

第三步： 设置 real_ip_recursive on;
至此，已经成功识别真实客户端ip，并赋予变量给了$remote_addr，接下来就可以针对这个去限制访问请求速率。

### 限制访问速率

这部分资料已经很多了，这里直接给配置文件示例：
```
http {
  # 定义2条限速区域
  # 以 $binary_remote_addr 字段作为限频统计点
  # zone=one:10m # 定义区域名称为one，限统计总占用内存10MB
  # 限制请求频率为 最多1次/per second
  limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;

  # 以 $binary_remote_addr 作为统计限频的依据
  # zone=zone2:32m zone名称定为zone2，定义32MB的统计内存占用
  # 限制请求频率 最多10次/per minute
  limit_req_zone $binary_remote_addr zone=zone2:32m rate=10r/m;

  # Sets the desired logging level for cases when the server refuses to
  # process requests due to rate exceeding, or delays request processing.
  # Logging level for delays is one point less than for refusals;
  # for example, if “limit_req_log_level notice” is specified, delays are logged with the info level.
  limit_req_log_level info; # default error, <info | notice | warn | error>

  limit_req_status 503; # default 503 added from nginx-1.3.15

  server {
    # ...

    location / {
      # 使用zone=one的限频规则
      limit_req zone=one burst=5;
      # limit_req zone=zone2 burst=5 nodelay; # 使用
      proxy_pass http://1.1.1.1:8090;
    }
}
```

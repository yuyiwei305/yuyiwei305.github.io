---
title: 记录nginx更换淘宝tengine实现动态域名解析以及长连接解决方案
date: 2020-05-14 17:52:39
tags: [nginx,tengine,keepalive,dns]
---

### 问题描述

目前公司很多服务会通过nginx调用到外部服务，业务量起来的时候，通过查看上游日志，可以看到有大量的连接产生，随着连接数增加会给服务器带来巨大压力。
所以针对这个问题去做一些性能优化。
优化项如下：

1. 更换基础nginx镜像为淘宝tengine镜像 --- tengine 支持upstream同时还支持域名动态解析，具体模块为： ngx_http_upstream_keepalive_module，ngx_http_upstream_dynamic_module。由于没有官方提供的tengine docker镜像 需要自己编译打包。
2. 修改pod dns选项优化查询域。
3. 配置文件参数调优,主要针对 https，以及websocket。

<!-- more -->


### 执行细节

##### 1. 编译tengine Dockerfile
这里没啥说的，一点一点趟坑搞出来的，直接给出Dockerfile:
```
FROM debian:buster-slim
LABEL maintainer="VisIon  <yuyiwei@bituniverse.org>"

ENV TENGINE_VERSION 2.3.2
ENV CONFIG "\
        --user=nginx \
        --group=nginx \
        --prefix=/etc/nginx \
        --sbin-path=/usr/sbin/nginx \
        --conf-path=/etc/nginx/nginx.conf \
        --lock-path=/var/lock/nginx.lock \
        --pid-path=/var/run/nginx.pid \
        --error-log-path=/var/log/nginx/error.log \
        --http-log-path=/var/log/nginx/access.log \
        --http-client-body-temp-path=/var/cache/nginx/client_temp \
        --http-proxy-temp-path=/var/cache/nginx/proxy_temp \
        --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
        --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
        --http-scgi-temp-path=/var/cache/nginx/scgi_temp \
        --with-http_ssl_module \
        --with-http_gzip_static_module \
        --with-http_gunzip_module \
        --with-http_auth_request_module \
        --with-http_image_filter_module \
        --with-http_addition_module \
        --with-http_dav_module \
        --with-http_realip_module \
        --with-http_v2_module \
        --with-http_stub_status_module \
        --with-http_sub_module \
        --with-http_xslt_module \
        --with-http_flv_module \
        --with-http_mp4_module \
        --with-http_degradation_module \
        --add-module=modules/ngx_http_upstream_dynamic_module \
        "

RUN set -x \
    && addgroup --system --gid 101 nginx \
    && adduser --system --disabled-login --ingroup nginx --no-create-home --home /nonexistent --gecos "nginx user" --shell /bin/false --uid 101 nginx \
    && apt-get update \
    && apt-get -y install \
       gcc \
       libxslt-dev \
       libxml2-dev \
       libc-dev \
       make \
       linux-libc-dev \
       curl \
       libpcre3 \
       libpcre3-dev \
       openssl \
       libssl-dev \
       zlib1g  \
       zlib1g-dev \
       libgd-dev \
       libjemalloc-dev \
       libjemalloc2 \
       curl \
       net-tools \
       procps \

    && curl "http://tengine.taobao.org/download/tengine-$TENGINE_VERSION.tar.gz" -o tengine.tar.gz \
    && mkdir -p /usr/src \
    && mkdir -p /var/cache/nginx \
    && chown nginx:nginx /var/cache/nginx \
    && tar -zxC /usr/src -f tengine.tar.gz \
    && rm tengine.tar.gz \
    && cd /usr/src/tengine-$TENGINE_VERSION \
    && ./configure $CONFIG \
    && make \
    && make install \
    && rm -rf /etc/nginx/html/ \
    && mkdir /etc/nginx/conf.d/ \
    && mkdir -p /usr/share/nginx/html/  \
    && install -m644 html/index.html /usr/share/nginx/html/ \
    && install -m644 html/50x.html /usr/share/nginx/html/ \
    && strip /usr/sbin/nginx* \
    && cd /etc/nginx \
    && rm -rf /usr/src/nginx-$NGINX_VERSION \
    && ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log

COPY nginx.conf /etc/nginx/nginx.conf
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]

```

default.conf 没啥说的 nginx.conf 因人而异。我这里给出我的：
nginx.conf

```
user nginx;
worker_processes auto;
pid /var/run/nginx.pid;
worker_rlimit_nofile 16384;

events {
  worker_connections 8192;
}

http {
  server_names_hash_bucket_size 128;
  map $http_upgrade $connection_upgrade {
      default upgrade;
      '' close;
  }
  log_format main
      '$remote_addr $host $remote_user [$time_local] "$request" $status $bytes_sent "$http_referer" "$http_user_agent" $request_time $upstream_response_time $upstream_addr';

  client_header_timeout  300s; # If after this time the client send nothing, nginx returns error "Request time out" (408).
  client_body_timeout    300s;
  send_timeout           300s; # if after this time client will take nothing, then nginx is shutting down the connection.
  proxy_connect_timeout  10s;
  proxy_read_timeout     10s;
  proxy_send_timeout     10s;
  connection_pool_size        256;
  client_header_buffer_size    1k;
  large_client_header_buffers    4 2k;
  request_pool_size        4k;
  output_buffers   4 32k;
  postpone_output  1460;
  sendfile        on;
  tcp_nopush             on;
  keepalive_timeout      60 30;
  tcp_nodelay            on;
  real_ip_header X-Forwarded-For;
  set_real_ip_from 0.0.0.0/0;

  client_max_body_size       10m;
  client_body_buffer_size    256k;

  gzip on;
  gzip_min_length  1100;
  gzip_comp_level  4;
  gzip_buffers     4 32k;
  gzip_types       application/json text/plain application/x-javascript text/xml text/css;

  ignore_invalid_headers    on;
  resolver 172.20.0.10 valid=10s ipv6=off;  # only valid in bthub-vpc oregon
  limit_req_status 555;                  # 将受限制的请求错误码转换为555
  proxy_ssl_server_name on;     # 针对SNI的https保证能通


  access_log /var/log/nginx/access.log main;
  error_log /var/log/nginx/error.log;

  include /etc/nginx/conf.d/*.conf;
  include /etc/nginx/sites-enabled/*;
}

```

至此tengine 2.3.2 镜像已经制作完成。 使用debian:buster-slim 作为基础镜像原因就是它支持libc。

##### 2. 优化查询域，这个是k8s里需要在deployment 改动的地方，主要的做法是，把默认的ndots 改小。

这里给出demo
```
dnsConfig:
        options:
        - name: single-request-reopen
        - name: ndots
          value: "4"
      dnsPolicy: ClusterFirst
```
其中 single-request-reopen  这个参数 alpine 是不支持的。 所以采用了 debian:buster-slim

##### 3. 配置趟坑

主要是业务的配置

```
upstream pionex-broker {
   dynamic_resolve fallback=stale fail_timeout=30s;
   server xxx.com:443;
   keepalive 32;
   keepalive_timeout 30s; # 设置后端连接的最大idle时间为30s
}
```

这里如果使用的是https,上游后面一定要跟 端口 ---> 443 否则会报错。



到此问题已经解决使用了 upstream  keepalive 和 dynamic_resolve 可以有效解决连接数过大导致性能压力问题。
经过测试，连接数由原来的几万瞬间降低到几百，延迟也由原来十几毫秒降低到几毫秒。

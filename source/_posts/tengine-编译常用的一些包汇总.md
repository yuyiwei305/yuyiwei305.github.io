---
title: tengine 编译常用的一些包汇总
date: 2020-09-27 21:00:52
tags: [tengine,nginx]
---
### 说明
tengine 配合lua性能没话说,但有时候会遇到很多包和库需要自己编译下载，这里列出常用的以备不时之需。

1、官方源码包
wget http://tengine.taobao.org/download/tengine-2.3.0.tar.gz



2、基础依赖库
http://luajit.org/download/LuaJIT-2.0.5.tar.gz
https://www.openssl.org/source/openssl-1.0.2q.tar.gz
https://github.com/maxmind/geoip-api-c/releases/download/v1.6.12/GeoIP-1.6.12.tar.gz
https://sourceforge.net/projects/pcre/files/pcre/8.42/pcre-8.42.tar.gz
https://github.com/openresty/lua-cjson/archive/2.1.0.6.tar.gz
https://github.com/libgd/libgd/releases/download/gd-2.2.5/libgd-2.2.5.tar.gz
https://github.com/yaoweibin/nginx_tcp_proxy_module/archive/v0.4.5.zip

3、第三方模块
1）需静态编译的C模块
https://github.com/simplresty/ngx_devel_kit/archive/v0.3.0.tar.gz
https://github.com/openresty/array-var-nginx-module/archive/v0.05.tar.gz
https://github.com/calio/form-input-nginx-module/archive/v0.12.tar.gz
https://github.com/openresty/encrypted-session-nginx-module/archive/v0.08.tar.gz
https://github.com/calio/iconv-nginx-module/archive/v0.14.tar.gz
https://github.com/openresty/lua-nginx-module/archive/v0.10.13.tar.gz
https://github.com/openresty/set-misc-nginx-module/archive/v0.32.tar.gz

<!-- more -->

2）通过DSO动态加载的C模块
https://github.com/openresty/echo-nginx-module/archive/v0.61.tar.gz
https://github.com/openresty/headers-more-nginx-module/archive/v0.33.tar.gz
https://github.com/openresty/lua-upstream-nginx-module/archive/v0.07.tar.gz
https://github.com/openresty/memc-nginx-module/archive/v0.19.tar.gz
https://github.com/vozlt/nginx-module-vts/archive/v0.1.18.tar.gz
https://github.com/weibocom/nginx-upsync-module/archive/v1.0.0.tar.gz
https://github.com/FRiCKLE/ngx_coolkit/archive/0.2.tar.gz
https://github.com/openresty/rds-csv-nginx-module/archive/v0.09.tar.gz
https://github.com/openresty/rds-json-nginx-module/archive/v0.15.tar.gz
https://github.com/openresty/redis2-nginx-module/archive/v0.15.tar.gz
https://github.com/openresty/srcache-nginx-module/archive/v0.31.tar.gz

3）第三方lua模块
https://github.com/hamishforbes/lua-resty-consul/archive/v0.3.1.tar.gz
https://github.com/cloudflare/lua-resty-cookie/archive/v0.1.0.tar.gz
https://github.com/openresty/lua-resty-core/archive/v0.1.15.tar.gz
https://github.com/openresty/lua-resty-dns/archive/v0.21.tar.gz
https://github.com/ledgetech/lua-resty-http/archive/v0.12.tar.gz
https://github.com/hamishforbes/lua-resty-iputils/archive/v0.3.0.tar.gz
https://github.com/doujiang24/lua-resty-kafka/archive/v0.06.tar.gz
https://github.com/upyun/lua-resty-limit-rate/archive/v0.1.tar.gz
https://github.com/openresty/lua-resty-limit-traffic/archive/v0.05.tar.gz
https://github.com/openresty/lua-resty-lock/archive/v0.07.tar.gz
https://github.com/cloudflare/lua-resty-logger-socket/archive/v0.1.tar.gz
https://github.com/openresty/lua-resty-lrucache/archive/v0.08.tar.gz
https://github.com/openresty/lua-resty-memcached/archive/v0.14.tar.gz
https://github.com/openresty/lua-resty-mysql/archive/v0.21.tar.gz
https://github.com/openresty/lua-resty-redis/archive/v0.26.tar.gz
https://github.com/bungle/lua-resty-session/archive/v2.23.tar.gz
https://github.com/openresty/lua-resty-string/archive/v0.11.tar.gz
https://github.com/bungle/lua-resty-template/archive/v1.9.tar.gz
https://github.com/openresty/lua-resty-upload/archive/v0.10.tar.gz
https://github.com/hamishforbes/lua-resty-upstream/archive/v0.09.tar.gz
https://github.com/openresty/lua-resty-upstream-healthcheck/archive/v0.05.tar.gz
https://github.com/openresty/lua-resty-websocket/archive/v0.06.tar.gz


###参考链接
https://www.cnblogs.com/grape-lee/p/10945356.html

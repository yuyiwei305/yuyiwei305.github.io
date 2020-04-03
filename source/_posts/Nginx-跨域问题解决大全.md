---
title: Nginx 跨域问题解决大全
date: 2020-04-02 11:33:47
tags: [nginx, 跨域]
---
工作中经常会遇到 No 'Access-Control-Allow-Origin' header is present on the requested resource 这样的跨域错误，此时需要给Nginx服务器配置响应的header参数：

# 一般解决方案

修改nginx配置文件：
```
location / {  
    add_header Access-Control-Allow-Origin "*";
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
    add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';

    if ($request_method = 'OPTIONS') {
        return 204;
    }
}
```

<!-- more -->

如果有不好使的情况 可以尝试在后面添加always参数:
```
location / {  
    add_header Access-Control-Allow-Origin "*" always;
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS' always ;
    add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization' always ;

    if ($request_method = 'OPTIONS') {
        return 204;
    }
}
```
# 跨域多个域名

一般来说工作场景中不会允许所有域名跨域，很多时候我们需要对指定的几个域名允许跨域请求,所以可以采用如下方式
```
location / {  
set $cors_origin "";
        if ($http_origin ~* "^http://test.blyoo.com$") {
                set $cors_origin $http_origin;
        }
        if ($http_origin ~* "^https://www.blyoo.com$") {
                set $cors_origin $http_origin;
        }
        add_header Access-Control-Allow-Origin $cors_origin;
        add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
        add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';
}
```

如果还不好使请尝试加一个header Vary:
```
location / {  
set $cors_origin "";
        if ($http_origin ~* "^http://test.blyoo.com$") {
                set $cors_origin $http_origin;
        }
        if ($http_origin ~* "^https://www.blyoo.com$") {
                set $cors_origin $http_origin;
        }
        add_header Access-Control-Allow-Origin $cors_origin;
        add_header Vary: Origin;
        add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
        add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';
}
```

至此nginx绝大部分跨域问题就基本解决了。

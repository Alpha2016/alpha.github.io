---
layout:     post
title:      Nginx 转发 socket 端口配置
subtitle:   Nginx 转发 socket 端口配置
date:       2019-06-03
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - Nginx
    - socket 请求
    - hop-by-hop
    - proxy_set_header 
    - proxy_pass
---

> Nginx 转发 socket 端口常见场景：在线学习应用，在常规功能之外，增加一个聊天室功能，后端选择 swoole 提供服务提供者，同时不想前端直接 ip:port 方式链接到服务，需要使用 Nginx 进行转发。

常规情况，我们可以在用户页面，直接建立 socket 链接，但这样的操作会暴露端口，带来一定的安全隐患，使用 Nginx 进行转发，可以隐藏端口。额外的问题就是一些 header 参数也需要在转发过程中带给 socket 服务提供者，其他只需要 Nginx 处理一下从常规协议转换到 Websocket 就可以。

其中，"Upgrade" 是 **逐跳（hop-by-hop）** 头，无法从客户端转发到代理服务器，通过转发代理，客户端可以使用 CONNECT 方法来规避此问题。但是，这不适用于反向代理，因为客户端不知道任何代理服务器，并且需要在代理服务器上进行特殊处理。同时逐跳头包含 "Upgrade" 和 "Connection" 都无法传递，则需要在转换为 Websocket 的时候带上这两个参数：例如：

```nginx
location /chat/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

进阶：让转发到代理服务器的 "Connection" 头字段的值，取决于客户端请求头的 "Upgrade" 字段值。例如：
```nginx
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    server {
        ...

        location /chat/ {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
        }
    }
```

**注意：示例中的 http://backend 为一组负载均衡的服务器，只有单台服务器的，可以写成 `proxy_pass http://127.0.0.1:9501;` 这样的。**

此外，默认情况下，在 60 秒内未传送任何数据的链接将被关闭，时间可以使用 `proxy_read_timeout` 指令来延长。或者代理服务器可以配置定时发送 ping 帧来重置超时及检查链接是否可用。


参考链接： [Nginx Websocket proxying](http://nginx.org/en/docs/http/websocket.html)

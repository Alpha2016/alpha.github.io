---
layout:     post
title:      Nginx 作为http负载均衡器
subtitle:   Nginx 作为http负载均衡器
date:       2019-05-18
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - Nginx
    - http 负载均衡
    - Nginx 1.15
    - ip_hash
    - weight
    - least_conn
---

> 翻译自 [官方文档](https://nginx.org/en/docs/http/load_balancing.html)，以 Nginx 1.15 版本为准。

![Nginx 负载均衡](https://alpha2016.github.io/img/2019-05-18-nginx-load-balancer.png)

##### 介绍

跨多个应用程序实例的负载平衡是一种常用技术，用于优化资源利用率，最大化吞吐量，减少延迟并确保容错配置。

可以使用 nginx 作为非常有效的 HTTP 负载平衡器，将流量分配到多个应用程序服务器，并使用 nginx 提高 Web 应用程序的性能，可伸缩性和可靠性。

##### 负载均衡方法

nginx支持以下负载平衡机制\方法：

- round-robin - 对应用程序服务器的请求以循环方式分发，
- least-connected - 下一个请求被分配给活动连接数最少的服务器，
- ip-hash - 哈希函数用于确定应为下一个请求选择哪个服务器（基于客户端的IP地址）。

##### 默认负载均衡配置

使用nginx进行负载平衡的最简单配置可能如下所示：

```nginx
http {
    upstream myapp1 {
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://myapp1;
        }
    }
}
```

在上面的示例中，在 srv1-srv3 上运行了 3 个相同应用程序的实例。如果未特别配置负载平衡方法，则默认为循环。所有请求都代理到服务器组 myapp1，nginx 应用 HTTP 负载平衡来分发请求。

nginx 中的反向代理实现包括 HTTP，HTTPS，FastCGI，uwsgi，SCGI，memcached 和 gRPC 的负载平衡。

要为 HTTPS 而不是 HTTP 配置负载平衡，只需使用 "https" 作为协议。

为 FastCGI，uwsgi，SCGI，memcached 或 gRPC 设置负载平衡时，分别使用 [fastcgi_pass](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_pass)， [uwsgi_pass](https://nginx.org/en/docs/http/ngx_http_uwsgi_module.html#uwsgi_pass)， [scgi_pass](https://nginx.org/en/docs/http/ngx_http_scgi_module.html#scgi_pass)， [memcached_pass](https://nginx.org/en/docs/http/ngx_http_memcached_module.html#memcached_pass) 和 [grpc_pass](https://nginx.org/en/docs/http/ngx_http_grpc_module.html#grpc_pass) 指令。

##### 最小连接负载平衡

另一个负载平衡规则是最少连接的。在某些请求需要更长时间才能完成的情况下，最小连接允许更公平地控制应用程序实例上的负载。

使用最少连接的负载平衡，nginx 将尝试不会使繁忙的应用程序服务器超载请求过多，而是将新请求分发给不太繁忙的服务器。

当 [least_conn](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#least_conn) 指令用作服务器组配置的一部分时，将激活 nginx 中的最小连接负载平衡：
```nginx
    upstream myapp1 {
        least_conn;
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }
```

##### 会话持久性

请注意，通过循环或最少连接的负载平衡，每个后续客户端的请求可能会分发到不同的服务器。无法保证同一客户端始终指向同一服务器。

如果需要将客户端绑定到特定的应用程序服务器 - 换句话说，就始终尝试选择特定服务器而言，使客户端的会话 "粘滞" 或 "持久" - ip-hash 负载平衡机制可以是用过的。

使用 ip-hash，客户端的 IP 地址将用作散列密钥，以确定应为客户端的请求选择服务器组中的哪个服务器。此方法可确保来自同一客户端的请求始终定向到同一服务器，但此服务器不可用时除外。

要配置 ip-hash 负载平衡，只需将 [ip_hash](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#ip_hash) 指令添加 到服务器（上游）组配置：
```nginx
upstream myapp1 {
    ip_hash;
    server srv1.example.com;
    server srv2.example.com;
    server srv3.example.com;
}
```


##### 加权负载平衡

通过使用服务器权重，甚至可以进一步影响 nginx 负载平衡算法。

在上面的示例中，未配置服务器权重，这意味着所有指定的服务器都被视为对特定负载平衡方法具有同等资格。

特别是循环，它还意味着在服务器上或多或少地平等分配请求 - 只要有足够的请求，并且以统一的方式处理请求并且足够快地完成。

当为服务器指定 [weight](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 参数时， 权重被计入负载平衡决策的一部分。
```nginx
upstream myapp1 {
    server srv1.example.com weight=3;
    server srv2.example.com;
    server srv3.example.com;
}
```

使用此配置，每5个新请求将分布在应用程序实例中，如下所示：3个请求将定向到 srv1，一个请求将转到 srv2，另一个请求转到 srv3。

同样可以在最近版本的 nginx 中简单将 weight，least-connected 和 ip-hash 一起来当作负载平衡配置。

##### 健康检查

nginx 中的反向代理实现包括频内（或被动）服务器运行状况检查。如果来自特定服务器的响应失败并出现错误，则 nginx 会将此服务器标记为失败，并将尝试避免为后续入站请求选择此服务器一段时间。

该 [max_fails](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 指令设置连续不成功的尝试与中应该发生的服务器进行通信的数量 [fail_timeout](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server)。默认情况下， [max_fails](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 设置为 1.当它设置为 0时 ，将禁用此服务器的运行状况检查。该 [fail_timeout](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 参数还定义如何，只要服务器失败将被标记。在 服务器发生故障后的 [fail_timeout](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 间隔后，nginx 将开始使用实时客户端的请求正常探测服务器。如果探测成功，则将服务器标记为实时。

##### 进一步阅读

此外，还有更多指令和参数可以控制 nginx 中的服务器负载平衡，例如 proxy_next_upstream， backup， down 和 keepalive。有关更多信息，请查看我们的[参考文档](https://nginx.org/en/docs/)。

最后但同样重要的是，[应用程序负载平衡](https://www.nginx.com/products/application-load-balancing/)，[应用程序运行状况检查](https://www.nginx.com/products/application-health-checks/)， [活动监视](https://www.nginx.com/products/live-activity-monitoring/) 和 服务器组的[动态重新配置](https://www.nginx.com/products/on-the-fly-reconfiguration/) 是我们付费 NGINX Plus 订阅的一部分。

以下文章更详细地描述了与NGINX Plus的负载平衡：
- [使用NGINX和NGINX Plus进行负载均衡](https://www.nginx.com/blog/load-balancing-with-nginx-plus/)
- [使用NGINX和NGINX Plus第2部分进行负载均衡](https://www.nginx.com/blog/load-balancing-with-nginx-plus-part2/)


最后恰饭 [阿里云全系列产品/短信包特惠购买 中小企业上云最佳选择 阿里云内部优惠券](https://www.aliyun.com/minisite/goods?userCode=0amqgcs9)
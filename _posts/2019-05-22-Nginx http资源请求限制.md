---
layout:     post
title:      Nginx http资源请求限制
subtitle:   Nginx http资源请求限制
date:       2019-05-22
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - Nginx
    - http 请求限制
    - 限制链接数
    - 限制请求速率
    - 限制带宽
---

> 前置条件：nginx 需要有 `ngx_http_limit_conn_module` 和 `ngx_http_limit_req_module` 模块，可以使用命令 `2>&1 nginx -V | tr ' '  '\n'|grep limit` 检查有没有相应模块，
如果没有请重新编译安装这两个模块。

**测试版本为：nginx版本为1.15+**

##### 限制链接数
1. 使用 `limit_conn_zone` 指令定义密钥并设置共享内存区域的参数（工作进程将使用此区域来共享密钥值的计数器）。第一个参数指定作为键计算的表达式。第二个参数 `zone` 指定区域的名称及其大小：
```nginx
limit_conn_zone $binary_remote_addr zone=addr:10m;
```
2. 在 `location {}`, `server {}` 或者 `http {}` 上下文中使用 `limit_conn` 指令来应用限制，第一个参数为上面设定的共享内存区域名称，第二个参数为每个key被允许的链接数:
```nginx
location /download/ {
     limit_conn addr 1;
}
```
使用 ` $binary_remote_addr` 变量作为参数的时候，是基于 IP 地址的限制，同样可以使用 `$server_name` 变量进行给定服务器连接数的限制：
```nginx
http {
    limit_conn_zone $server_name zone=servers:10m;

    server {
        limit_conn servers 1000;
    }
}
```

##### 限制请求速率
速率限制可用于防止 DDoS，CC 攻击，或防止上游服务器同时被太多请求淹没。该方法基于 `leaky bucket` 漏桶算法，请求以各种速率到达桶并以固定速率离开桶。在使用速率限制之前，您需要配置 "漏桶" 的全局参数：
- key - 用于区分一个客户端与另一个客户端的参数，通常是变量
- shared memory zone - 保留这些密钥状态的区域的名称和大小（即 "漏桶"）
- rate - 每秒请求数（r/s）或每分钟请求数（r/m）（"漏桶排空"）中指定的请求速率限制。每分钟请求数用于指定小于每秒一个请求的速率。
这些参数使用 `limit_req_zone` 指令设置。该指令在 `http {}` 级别上定义 - 这种方法允许应用不同的区域并请求溢出参数到不同的上下文:
```nginx
http {
    #...

    limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
}
```
使用此配置，将创建大小为 10m 字节，名称为 one 的共享内存区域。该区域保存使用 `$binary_remote_addr` 变量设置的客户端 IP 地址的状态。请注意，`$remote_addr` 还包含客户端的 IP 地址，而 `$binary_remote_addr` 保留更短的 IP 地址的二进制表示。

可以使用以下数据计算共享内存区域的最佳大小：`$binary_remote_addr` IPv4 地址的值大小为 4 个字节，64 位平台上的存储状态占用 128 个字节。因此，大约 16000 个 IP 地址的状态信息占用该区域的 1m 字节。

如果在 NGINX 需要添加新条目时存储空间耗尽，则会删除最旧的条目。如果释放的空间仍然不足以容纳新记录，NGINX 将返回 `503 Service Unavailable` 状态代码，状态码可以使用 `limit_req_status` 指令重新定义。

一旦该区域被设置，你可以使用 NGINX 配置中的任何地方使用 `limit_req` 指令限制请求速率，尤其是 `server {}`, `location {}` 和 `http {}` 上下文：
```nginx
http {
    #...

    limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;

    server {
        #...

        location /search/ {
            limit_req zone=one;
        }
    }
}
```
使用如上配置，nginx 在 `/search/` 路由下将每秒处理不超过 1 个请求，延迟处理这些请求的方式是总速率不大于设定的速率。NGINX 将延迟处理此类请求，直到 "存储区"（共享存储区 one）已满。对于到达完整存储桶的请求，NGINX 将响应 `503 Service Unavailable` 错误(当 `limit_req_status` 未自定义设定状态码时)。


##### 限制宽带
要限制每个连接的带宽，请使用以下 `limit_rate` 指令：
```nginx
location /download/ {
    limit_rate 50k;
}
```
通过此设置，客户端将能够通过单个连接以最高 50k/秒 的速度下载内容。但是，客户端可以打开多个连接跳过此限制。因此，如果目标是阻止下载速度大于指定值，则连接数也应该受到限制。例如，每个 IP 地址一个连接（如果使用上面指定的共享内存区域）:
```nginx
location /download/ {
    limit_conn addr 1;
    limit_rate 50k;
}
```
要仅在客户端下载一定数量的数据后施加限制，请使用该 `limit_rate_after` 指令。允许客户端快速下载一定数量的数据（例如，文件头 - 电影索引）并限制下载其余数据的速率（使用户观看电影而不是下载）可能是合理的。
```nginx
limit_rate_after 500k;
limit_rate 20k;
```
以下示例显示了用于限制连接数和带宽的组合配置。允许的最大连接数设置为每个客户端地址 5 个连接，这适用于大多数常见情况，因为现代浏览器通常一次最多打开 3 个连接。同时，提供下载的位置只允许一个连接：
```nginx
http {
    limit_conn_zone $binary_remote_address zone=addr:10m

    server {
        root /www/data;
        limit_conn addr 5;

        location / {
        }

        location /download/ {
            limit_conn addr 1;
            limit_rate_after 1m;
            limit_rate 50k;
        }
    }
}
```

内容翻译自 [nginx 请求限制部分文档](https://docs.nginx.com/nginx/admin-guide/security-controls/controlling-access-proxied-http/)，稍微调整了一点语义。

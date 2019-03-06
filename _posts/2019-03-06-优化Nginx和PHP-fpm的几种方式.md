---
layout:     post
title:      优化Nginx及PHP-fpm的几种方式
subtitle:   优化Nginx及PHP-fpm的几种方式
date:       2019-03-06
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - PHP-fpm
    - Nginx
    - 优化方案
---

Nginx 和 PHP 的关系：<br />
![Nginx 和 PHP 的关系](https://alpha2016.github.io/img/2019-03-06-nginx-and-php.jpg "Nginx 和 PHP 的关系")

#### Nginx 优化
1. TCP 与 UNIX 套接字 <br />
UNIX 域套接字提供的性能略高于 TCP 套接字在回送接口上的性能（较少的数据复制，较少的上下文切换）。如果每个服务器需要支持超过 1000 个连接，请使用 TCP 套接字 - 它们可以更好地扩展。<br />
```text
upstream backend 
{ 
  # UNIX domain sockets 
  server unix:/var/run/fastcgi.sock; 

  # TCP sockets 
  # server 127.0.0.1:8080; 
}
```

2. 调整 worker_processes 参数<br />
现代硬件是多处理器，NGINX 可以利用多个物理或虚拟处理器。在大多数情况下，您的 Web 服务器计算机不会配置为处理多个工作负载（例如同时提供 Web 服务器和打印服务器的服务），因此您需要配置 NGINX 以使用所有可用的处理器，因为 NGINX 工作进程是不是多线程的。<br />
将 nginx.conf 文件中的 worker_processes 设置为计算机所具有的核心数。<br />
当你在它的时候，增加 worker_connections 的数量（每个核心应该处理多少个连接）并将 multi_accept 设置为 ON，如果你在 Linux 上则设置为 epoll：<br />
```text
# We have 4 cores 
worker_processes 4; 

# connections per worker 
events 
{ 
  worker_connections 1024; 
  multi_accept on; 
}
```

3. 禁用访问日志<br />
```text
access_log off; 
log_not_found off; 
error_log /var/log/nginx-error.log warn;
```
如果有需要不可以关闭，至少是缓存他们<br />
```text
access_log /var/log/nginx/access.log main buffer=16k;
```

4. 开启 gzip 压缩<br />
```text
gzip on; 
gzip_disable "msie6"; 
gzip_vary on; 
gzip_proxied any; 
gzip_comp_level 6; 
gzip_min_length 1100; 
gzip_buffers 16 8k; 
gzip_http_version 1.1; 
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
```

5. 缓存访问频次较高的文件<br />
```text
open_file_cache max=2000 inactive=20s; 
open_file_cache_valid 60s; 
open_file_cache_min_uses 5; 
open_file_cache_errors off;
```

6. 调整客户端超时<br />
```text
client_max_body_size 50M; 
client_body_buffer_size 1m; 
client_body_timeout 15; 
client_header_timeout 15; 
keepalive_timeout 2 2; 
send_timeout 15; 
sendfile on; 
tcp_nopush on; 
tcp_nodelay on;
```

7. 调整输出缓存<br />
```text
fastcgi_buffers 256 16k; 
fastcgi_buffer_size 128k; 
fastcgi_connect_timeout 3s; 
fastcgi_send_timeout 120s; 
fastcgi_read_timeout 120s; 
fastcgi_busy_buffers_size 256k; 
fastcgi_temp_file_write_size 256k; 
reset_timedout_connection on; 
server_names_hash_bucket_size 100;
```

8. 负载均衡策略<br />
Nginx 提供轮询（round robin）、用户 IP 哈希（client IP）和指定权重 3 种方式。<br />
默认情况下，Nginx 会为你提供轮询作为负载均衡策略。但是这并不一定能够让你满意。比如，某一时段内的一连串访问都是由同一个用户 Michael 发起的，那么第一次 Michael 的请求可能是 backend2，而下一次是 backend3，然后是 backend1、backend2、backend3…… 在大多数应用场景中，这样并不高效。当然，也正因如此，Nginx 为你提供了一个按照 Michael、Jason、David 等等这些乱七八糟的用户的 IP 来 hash 的方式，这样每个 client 的访问请求都会被甩给同一个后端服务器。<br />
```text
upstream backend {
  ip_hash;
  server unix:/var/run/php7.2-fpm.sock1 weight=100 max_fails=5 fail_timeout=5; 
  server unix:/var/run/php7.2-fpm.sock2 weight=100 max_fails=5 fail_timeout=5; 
}
```

9. 持续监控日志<br />
尤其是在调优最初期，在一个窗口<br />
```text
tail -f /var/log/nginx/error.log
```
在另外两个窗口分别：<br />
```text
tail -f /var/log/php-fpm/error.log
tail -f /var/log/php-fpm/www-error.log
```

#### PHP-fpm 优化



参考链接：
1. [Optimizing NGINX and PHP-fpm for high traffic sites](http://www.softwareprojects.com/resources/programming/t-optimizing-nginx-and-php-fpm-for-high-traffic-sites-2081.html "Optimizing NGINX and PHP-fpm for high traffic sites")

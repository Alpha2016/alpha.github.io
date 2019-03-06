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
**1. TCP 与 UNIX 套接字**<br />
UNIX 域套接字提供的性能略高于 TCP 套接字在回送接口上的性能（较少的数据复制，较少的上下文切换）。如果每个服务器需要支持超过 1000 个连接，请使用 TCP 套接字 - 它们可以更好地扩展。<br />
```text
upstream backend 
{
  server unix:/var/run/fastcgi.sock; 
}
```

**2. 调整 worker_processes 参数**<br />
现代硬件是多处理器，NGINX 可以利用多个物理或虚拟处理器。在大多数情况下，您的 Web 服务器计算机不会配置为处理多个工作负载（例如同时提供 Web 服务器和打印服务器的服务），因此您需要配置 NGINX 以使用所有可用的处理器，因为 NGINX 工作进程是不是多线程的。<br />
将 nginx.conf 文件中的 worker_processes 设置为计算机所具有的核心数。<br />
当你在它的时候，增加 worker_connections 的数量（每个核心应该处理多少个连接）并将 multi_accept 设置为 ON，如果你在 Linux 上则设置为 epoll：<br />
```text
worker_processes 4;
events 
{ 
  worker_connections 1024; 
  multi_accept on; 
}
```

**3. 禁用访问日志**<br />
```text
access_log off; 
log_not_found off; 
error_log /var/log/nginx-error.log warn;
```
如果有需要不可以关闭，至少是缓存他们<br />
```text
access_log /var/log/nginx/access.log main buffer=16k;
```

**4. 开启 gzip 压缩**<br />
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

**5. 缓存访问频次较高的文件**<br />
```text
open_file_cache max=2000 inactive=20s; 
open_file_cache_valid 60s; 
open_file_cache_min_uses 5; 
open_file_cache_errors off;
```

**6. 调整客户端超时**<br />
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

**7. 调整输出缓存**<br />
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

**8. 负载均衡策略**<br />
Nginx 提供轮询（round robin）、用户 IP 哈希（client IP）和指定权重 3 种方式。<br />
默认情况下，Nginx 会为你提供轮询作为负载均衡策略。但是这并不一定能够让你满意。比如，某一时段内的一连串访问都是由同一个用户 Michael 发起的，那么第一次 Michael 的请求可能是 backend2，而下一次是 backend3，然后是 backend1、backend2、backend3…… 在大多数应用场景中，这样并不高效。当然，也正因如此，Nginx 为你提供了一个按照 Michael、Jason、David 等等这些乱七八糟的用户的 IP 来 hash 的方式，这样每个 client 的访问请求都会被甩给同一个后端服务器。<br />
```text
upstream backend {
  ip_hash;
  server unix:/var/run/php7.2-fpm.sock1 weight=100 max_fails=5 fail_timeout=5; 
  server unix:/var/run/php7.2-fpm.sock2 weight=100 max_fails=5 fail_timeout=5; 
}
```

**9. 持续监控日志**<br />
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

**1. 修改进程管理模式**<br />
static 管理模式适合比较大内存的服务器，而 dynamic 则适合小内存的服务器，你可以设置一个 pm.min_spare_servers 和 pm.max_spare_servers 合理范围，这样进程数会不断变动。ondemand模式则更加适合微小内存，例如 512MB 或者 256MB 内存，以及对可用性要求不高的环境。<br />
```apacheconfig
pm = dynamic #指定进程管理方式，有3种可供选择：static、dynamic和ondemand。
pm.max_children = 16 #static模式下创建的子进程数或dynamic模式下同一时刻允许最大的php-fpm子进程数量。
pm.start_servers = 10 #动态方式下的起始php-fpm进程数量。
pm.min_spare_servers = 8 #动态方式下服务器空闲时最小php-fpm进程数量。
pm.max_spare_servers = 16 #动态方式下服务器空闲时最大php-fpm进程数量。
pm.max_requests = 2000 #php-fpm子进程能处理的最大请求数。
pm.process_idle_timeout = 10s
request_terminate_timeout = 120
```
pm = static，始终保持一个固定数量的子进程，这个数由 pm.max_children 定义，这种方式很不灵活，也通常不是默认的。<br />
pm = dynamic，启动时会产生固定数量的子进程（由 pm.start_servers 控制）可以理解成最小子进程数，而最大子进程数则由 pm.max_children 去控制，子进程数会在最大和最小数范围中变化。闲置的子进程数还可以由另2个配置控制，分别是 pm.min_spare_servers 和 pm.max_spare_servers。如果闲置的子进程超出了pm.max_spare_servers，则会被杀掉。小于 pm.min_spare_servers 则会启动进程（注意，pm.max_spare_servers 应小于 pm.max_children）。<br />
pm = ondemand，这种模式和 pm = dynamic 相反，把内存放在第一位，每个闲置进程在持续闲置了 pm.process_idle_timeout 秒后就会被杀掉，如果服务器长时间没有请求，就只会有一个 php-fpm 主进程。弊端是遇到高峰期或者如果 pm.process_idle_timeout 的值太短的话，容易出现 504 Gateway Time-out 错误，因此 pm = dynamic 和 pm = ondemand 谁更适合视实际情况而定。<br />

**2. 释放内存的配置**<br />
```apacheconfig
pm.max_requests = 1000
```
设置每个子进程重生之前服务的请求数. 对于可能存在内存泄漏的第三方模块来说是非常有用的. 如果设置为 '0' 则一直接受请求. 等同于 PHP_FCGI_MAX_REQUESTS 环境变量. 默认值: 0.<br />
也就是当一个 PHP-CGI 进程处理的请求数累积到1000个后，自动重启该进程，防止第三方库造成的内存泄漏。
重启时可能会导致502错误，在高并发站点时有出现。<br />

**3. php-fpm 慢日志**<br />
```apacheconfig
request_terminate_timeout = 30s #将执行时间太长的进程直接终止
request_slowlog_timeout = 2s #2秒
slowlog = log/$pool.log.slow #日志文件
```

参考链接：
1. [Optimizing NGINX and PHP-fpm for high traffic sites](http://www.softwareprojects.com/resources/programming/t-optimizing-nginx-and-php-fpm-for-high-traffic-sites-2081.html "Optimizing NGINX and PHP-fpm for high traffic sites")
2. [PHP-fpm 文章](https://www.zybuluo.com/phper/note/89081)
3. [A better way to run PHP-FPM](https://ma.ttias.be/a-better-way-to-run-php-fpm/ "A better way to run PHP-FPM")
4. [PHP-fpm 配置官方文档](http://php.net/manual/zh/install.fpm.configuration.php "PHP-fpm 配置官方文档")

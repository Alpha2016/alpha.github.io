---
layout:     post
title:      Nginx gzip压缩静态文件
subtitle:   Nginx gzip压缩静态文件
date:       2019-05-14
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - PHP
    - Nginx
    - gzip
    - 压缩静态文件
---

优化网站加载速度方案里有一条：压缩静态资源，其中可以使用 Nginx gzip 模块进行压缩，配置为：

```nginx
#启用gzip压缩的 nginx 配置

gzip on;
 
gzip_min_length 1k;
 
gzip_buffers 4 16k;
 
#gzip_http_version 1.0;
 
gzip_comp_level 2;
 
gzip_types text/plain application/x-javascript text/css application/xml text/javascript application/x-httpd-php;
 
gzip_vary off;
 
gzip_disable "MSIE [1-6]\.";
```

**以上含义为：** <br />
gzip on 为启用 gzip 压缩<br />
gzip_min_length 压缩大小的临界值，小于 1k 不压缩 <br />
gzip_buffers 设置 number 和 size 用于压缩的响应缓冲区。默认情况下，缓冲区大小等于一个内存页面。这是 4K 或 8K ，取决于平台。<br />
gzip_comp_level 压缩级别，可选范围 1-9，越高则压缩质量越高，消耗 CPU 资源越高<br />
gzip_types 压缩的内容类型，需要什么再加*不要压缩图片*<br />
gzip_vary 跟 Squid 等缓存服务有关，on 的话会在 Header 里增加 "Vary: Accept-Encoding" <br />
gzip_disable 对某种指定的浏览器类型不启用压缩，M$ie6以下肯定不压缩了

**注意点：图片不适用于压缩，因为压缩效率很低，压缩之后不会减少多少体积，同时压缩过程会非常消耗 CPU 资源，得不偿失，图片类有其他的处理方式**

使用 Nginx 只是其中一种压缩静态资源的方式，除此之外，可以使用 gulp 等方式压缩前端资源，提前压缩，直接使用压缩后的资源，减少消耗。如果高流量访问的大型网站，直接静态资源 CDN 化是最好的选择，具体根据团队和项目情况来选择方案。

参考链接：[Nginx gzip module文档](https://github.com/DocsHome/nginx-docs/blob/master/%E6%A8%A1%E5%9D%97%E5%8F%82%E8%80%83/http/ngx_http_gzip_module.md)

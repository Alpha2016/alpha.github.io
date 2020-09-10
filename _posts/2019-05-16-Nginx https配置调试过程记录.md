---
layout:     post
title:      Nginx https配置调试过程记录
subtitle:   Nginx https配置调试过程记录
date:       2019-05-16
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - Centos7
    - Nginx
    - https 配置
    - 阿里云 防火墙规则
    - Nginx listen
---

> 测试环境为：阿里云 centos7.4 ，nginx1.14.3，其他版本的系统或者nginx如有不同，以官网为准。 

##### 开始配置

 最开始参考阿里云栖社区的这篇 [文章](https://yq.aliyun.com/articles/633435)，在阿里云控制面板进行配置，然后对应修改 `nginx.conf` 文件，执行 `nginx -s reload` 重载使之生效。

 *nginx.conf https配置*
```nginx
server {
    listen 443;
    server_name www.domain.com;
    ssl on;
    root html;
    index index.html index.htm;
    ssl_certificate   cert/domain.pem;
    ssl_certificate_key  cert/domain.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location / {
        root html;
        index index.html index.htm;
    }
}
```

一番操作之后域名 https://www.domain.com 不能访问，可怕了，开始排错。

##### 检查配置和本地端口
**检查域名，证书位置，其他参数有没有写错的** <br />
主要是看一下，没有发现错误。

**使用 `nginx -t` 测试配置**<br />
```shell
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
```
nginx 自身的测试没有报错。

**检查本机端口监听**<br />
使用 `netstat -anp |grep 443` 命令，检查443端口监听，结果：
`tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      15940/nginx: master` 也正常，所以问题出在外部网络端口上。

##### 检查防火墙端口及安全组规则
前提条件，使用 `telnet www.domain.com 443` 请求 443 端口，报错 443 端口无法链接，所以开放 443 端口可以访问。

**检查安全组设置** <br />
直接参考 [官方文档](https://www.alibabacloud.com/help/zh/doc-detail/25471.htm), 如果安全组没有 443 端口，加上并且允许访问就行，现在多少是默认开启的。

**firewall添加443端口** <br />
使用以下命令：
```shell
firewall-cmd --list-ports
#output 80/tcp
firewall-cmd --zone=public --add-port=443/tcp --permanent
firewall-cmd --reload
```
重载生效，理想情况下不会有什么问题了，然后浏览器访问还是无法连接。

在命令行下，执行 `curl https://www.domain.com` 检查情况，返回报错： 
```shell
curl https://www.domain.com
curl: (35) SSL received a record that exceeded the maximum permissible length.
```

直接谷歌报错信息，MD，nginx 高版本需要端口和 ssl 需要在一行，改为 `listen 443 ssl` 然后去掉 `ssl on` 这行，再次 `nginx -s reload` 生效，一切正常了。搜到文章说 `ssl on` 这行在 `nginx 1.15` 版本中会报错，没去验证了，高版本 nginx 将这两个写在一行就行。

最终配置为：
```nginx
server {
    listen 443 ssl;
    server_name www.domain.com;
    # 其他配置不变

    ...
}
```

参考链接：[官方文档](http://nginx.org/en/docs/http/configuring_https_servers.html)

© 原创文章


最后恰饭 [阿里云全系列产品/短信包特惠购买 中小企业上云最佳选择 阿里云内部优惠券](https://www.aliyun.com/minisite/goods?userCode=0amqgcs9)
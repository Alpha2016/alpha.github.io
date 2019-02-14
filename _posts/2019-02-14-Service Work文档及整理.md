---
layout:     post
title:      Service Worker 文档及整理
subtitle:   Service Worker笔记，前端优化
date:       2019-02-14
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: false
tags:
    - Service Worker
---

> 这个是周边同事 gy 偶然发现的，在查看 Angular 官网的时候，发现网络请求中有很多从 Service Worker 中取的文件缓存

![Networks](https://alpha2016.github.io/img/2019-02-14-service-worker.jpg "Service Worker")

&ensp;&ensp;Service worker是一个注册在指定源和路径下的事件驱动worker。它采用JavaScript控制关联的页面或者网站，拦截并修改访问和资源请求，细粒度地缓存资源。你可以完全控制应用在特定情形（最常见的情形是网络不可用）下的表现。

&ensp;&ensp;简单来说是，可以把 Service Worker 理解为一个介于客户端和服务器之间的一个代理服务器。他提供的功能有：
* 后台数据同步
* 响应来自其它源的资源请求
* 集中接收计算成本高的数据更新，比如地理位置和陀螺仪信息，这样多个页面就可以利用同一组数据
* 在客户端进行CoffeeScript，LESS，CJS/AMD等模块编译和依赖管理（用于开发目的）
* 后台服务钩子
* 自定义模板用于特定URL模式
* 性能增强，比如预取用户可能需要的资源，比如相册中的后面数张图片
   
&ensp;&ensp;目前看用以提前下载静态资源和缓存方面的应用比较多，可以算是 CDN 优化之外的一个手段。更多具体使用请参考文档，非常详细，目前浏览器的支持正在完善中

参考链接：
1. [Mdn web doc](https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API "Service Worker")
2. [Google developer doc](https://developers.google.com/web/fundamentals/primers/service-workers/ "Service Worker")
3. [掘金文章](https://juejin.im/post/5b06a7b3f265da0dd8567513 "Service Worker")
4. [Laras 百度实践教程](https://lavas.baidu.com/pwa/offline-and-cache-loading/service-worker/service-worker-debug "Service Worker")

---
layout:     post
title:      Edge Chromium 开启 h264 播放支持
subtitle:   Edge Chromium 开启 h264 播放支持
date:       2020-08-08
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - Edge
    - h264
---

1. 开启 Windows10 支持
安装 [hevc 视频扩展](https://www.microsoft.com/zh-cn/p/hevc-video-extensions-from-device-manufacturer/9n4wgh0z6vhq), 这个是免费的，可以直接安装，或安装 [完美解码](https://jm.wmzhe.com/)

2. 开始实验室解码支持
在 edge 地址栏输入 `edge://flags` ，然后搜索 video, 找到 `Hardware-accelerated video decode`, `Hardware-accelerated video encode` 两个选项，设置成 enable，重启浏览器就可以支持 h264 编码的视频播放了。

**chrome 始终不支持**


最后恰饭 [阿里云全系列产品/短信包特惠购买 中小企业上云最佳选择 阿里云内部优惠券](https://www.aliyun.com/minisite/goods?userCode=0amqgcs9)
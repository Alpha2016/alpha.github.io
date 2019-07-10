---
layout:     post
title:      使用 Redis HyperLogLog 统计 UV 数据
subtitle:   使用 Redis HyperLogLog 统计 UV 数据
date:       2019-07-11
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - redis
    - HyperLogLog
    - 统计 UV
---

> 大概使用场景是统计几个活动页面的单独和总UV数量，可以偏差0-1.5%

实现逻辑：前端对于登陆或者未登录用户生成一个唯一ID，例如可以是 `md5(ip+user-agent) + hashcode`, 同时也能区分一个路由器下的多个设备，UV 的生存时间根据需求来确定，一般情况下是到 24 小时过期，有些是活动结束己过期。多数方案会将这写唯一 ID 存到 `redis set` 中，这会导致 set 占用空间很大，同时不断的扩容操作也会带来内存波动。

Redis 给提供了另一种数据结构，是能够接受微小偏差的，同时尽可能的减少战役的空间，可以采用 HyperLogLog ，只有几个简单的命令：
```
pfadd activity-page1 uuid1
pfcount actitity-page1
pfmerge activity-page-all activity-page1acvitipage2 …
```

`pfadd` 是为 `key` 值添加一条数据，`pfcount` 是统计这个 `key` 值 `pfmger` 是将后面的几个 `key` 所有的值合并到指定`key`, `pfadd` 在插入重复值的时候会返回0，这个需要注意一点，`pfmerge` 自带去重统计功能。

相关：<br />
`Redis HyperLogLog` 是用来做基数统计的算法，`HyperLogLog` 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。

在 `Redis` 里面，每个 `HyperLogLog` 键只需要花费 12 KB 内存，就可以计算接近 `2^64` 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。但是，因为 `HyperLogLog` 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 `HyperLogLog` 不能像集合那样，返回输入的各个元素。

更多文章：https://blog.csdn.net/heiyeshuwu/article/details/41248379


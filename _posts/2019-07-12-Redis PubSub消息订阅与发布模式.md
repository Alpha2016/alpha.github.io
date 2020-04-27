---
layout:     post
title:      Redis PubSub 消息定义与发布模式
subtitle:   Redis PubSub 消息定义与发布模式
date:       2019-07-12
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - redis
    - HyperLogLog
    - 统计 UV
---

**Redis的PubSub 模式即：**

基于事件的系统中，`Pub/Sub` 是目前广泛使用的通信模型，它采用事件作为基本的通信机制，提供大规模系统所要求的松散耦合的交互模式：订阅者(如客户端)以事件订阅的方式表达出它有兴趣接收的一个事件或一类事件；发布者(如服务器)可将订阅者感兴趣的事件随时通知相关订阅者。

消息发布者，即 `publish` 客户端，无需独占链接，你可以在 `publish` 消息的同时，使用同一个 `redis-client` 链接进行其他操作（例如：`INCR` 等）

消息订阅者，即 `subscribe` 客户端，需要独占链接，即进行 `subscribe` 期间，`redis-client` 无法穿插其他操作，此时 `client` 以阻塞的方式等待 `publish` 端的消息；这一点很好理解，因此 `subscribe` 端需要使用单独的链接，甚至需要在额外的线程中使用。

**基本使用：**

主要是订阅某频道，接收消息，取消订阅，额外的功能是：正则模式订阅，类似订阅整个频道，还有相关的基础统计功能

消费者收到的消息是：
```json
{'pattern': None, 'type': 'subscribe','channel': 'codehole', 'data': 1L} {'pattern': None, 'type': 'message','channel': 'codehole', 'data': 'python comes'}
```

其中 `data` 是消息的内容，一个字符串。`channel` 这个也很明显，它表示当前订阅的主题名称。 `type` 它表示消息的类型，如果是一个普通的消息，那么类型就是 `message`，如果是控制消息，比如订阅指令的反馈，它的类型就是 `subscribe`，如果是模式订阅的反馈，它的类型就是 `psubscribe`，还有取消订阅指令的反馈 `unsubscribe` 和 `punsubscribe`。`pattern` 它表示当前消息是使用哪种模式订阅到的，如果是通过 `subscribe` 指令订阅的，那么这个字段就是空。


**获取消息可以：**

消费者可以通过轮询来获取消息，没有消息/接收不到则休眠一秒，也可以使用 listen 来阻塞监听消息来进行处理，官方文档：[pubsub](http://doc.redisfans.com/pub_sub/pubsub.html)

缺陷：

PubSub 是一种即发即弃的模式，生产者传递过来一个消息，Redis 会直接找到相应的消费者传递过去。如果一个消费者都没有，那么消息直接丢弃。如果开始有三个消费者，一个消费者突然挂掉了，生产者会继续发送消息，另外两个消费者可以持续收到消息。但是挂掉的消费者重新连上的时候，这断连期间生产者发送的消息，对于这个消费者来说就是彻底丢失了。

如果 Redis 停机重启，PubSub 的消息是不会持久化的，Redis 宕机就相当于一个消费者都没有，所有的消息直接被丢弃。**简单来说是没有消息的备份和恢复机制**。如果大量消息同时涌入，redis没有对应的性能提升方案和慢消息方案，如果消费者来不及消费，可能造成消息堆积。


如果你期望消息是持久的：

① subscribe端首先向一个Set集合中增加“订阅者ID”，此Set集合保存了“活跃订阅”者，订阅者ID标记每个唯一的订阅者，例如：sub:email,sub:web。此SET称为“活跃订阅者集合”

② subcribe端开启订阅操作，并基于Redis创建一个以“订阅者ID”为KEY的LIST数据结构，此LIST中存储了所有的尚未消费的消息。此LIST称为“订阅者消息队列”

③ publish端：每发布一条消息之后，publish端都需要遍历“活跃订阅者集合”，并依次向每个“订阅者消息队列”尾部追加此次发布的消息。

④ 到此为止，我们可以基本保证，发布的每一条消息，都会持久保存在每个“订阅者消息队列”中。

⑤ subscribe端，每收到一个订阅消息，在消费之后，必须删除自己的“订阅者消息队列”头部的一条记录。

⑥ subscribe端启动时，如果发现自己的“订阅者消息队列”有残存记录，那么将会首先消费这些记录，然后再去订阅。
 
简单总结是：在额外的 `set` 中记录所有订阅者，并创建以订阅者 id 为 key 的`list` 数据结构，发布者将每一条消息发布成功之后，记录进订阅者 ID 为 key 的`value` 中，订阅者消费成功消息后，删除此消息，此外订阅者断线重连的情况下，读取并消费自身 ID 为 key 的 list 的数据，然后再重新订阅发布者，这样保证数据顺序不会乱，也能保证消息被存储和正确消费。

顺带推荐 [质量很高的课程](https://hxd.best/2021/04/01/%E6%8E%A8%E8%8D%90%E5%87%A0%E4%B8%AA%E4%B8%8D%E9%94%99%E7%9A%84%E6%95%99%E7%A8%8B-%E6%9E%81%E5%AE%A2%E6%97%B6%E9%97%B4%E4%B8%93%E6%A0%8F/)， 欢迎扫码购买

部分内容参考自：[cnblog 文章](http://www.cnblogs.com/shihaiming/p/6054192.html?utm_source=itdadao&utm_medium=referral) 及 掘金小册 Redis 深度历险 作者：老钱
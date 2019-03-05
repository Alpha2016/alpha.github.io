---
layout:     post
title:      程序在Redis list异常崩溃时的处理方案
subtitle:   程序在Redis list异常崩溃时的处理方案
date:       2019-03-05
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - PHP
    - Redis List
    - 异常处理方案
---

> 题目是：用 Redis list 做消息队列，在取出消息的时候 Redis 宕机，程序上的业务逻辑没执行，怎样处理？先不去考虑 Redis 的异常处理及恢复，一般主从哨兵机制很难崩溃，暂时考虑程序端如何处理这种

> 方案：直接使用 Redis 官方的 `RPOPLPUSH / BRPOPLPUSH source destination` 命令，在读取消息的同时，将读取到的消息内容放到目标队列，然后当前进程进行消费数据，消费成功，则 `lrem destination 1 key` 在目标队列中删除刚才的消息内容。

如图：<br />
![Redis RPOPLPUSH 命令](https://alpha2016.github.io/img/2019-03-05-redis-list-rpoplpush-demo.jpg "Redis RPOPLPUSH 命令")

官方文档：<br />
**安全的队列** <br />
&ensp;&ensp;Redis通常都被用做一个处理各种后台工作或消息任务的消息服务器。 一个简单的队列模式就是：生产者把消息放入一个列表中，等待消息的消费者用 RPOP 命令（用轮询方式）， 或者用 BRPOP 命令（如果客户端使用阻塞操作会更好）来得到这个消息。然而，因为消息有可能会丢失，所以这种队列并是不安全的。例如，当接收到消息后，出现了网络问题或者消费者端崩溃了， 那么这个消息就丢失了。<br />
&ensp;&ensp;RPOPLPUSH (或者其阻塞版本的 BRPOPLPUSH） 提供了一种方法来避免这个问题：消费者端取到消息的同时把该消息放入一个正在处理中的列表。 当消息被处理了之后，该命令会使用 LREM 命令来移除正在处理中列表中的对应消息。<br />
&ensp;&ensp;另外，可以添加一个客户端来监控这个正在处理中列表，如果有某些消息已经在这个列表中存在很长时间了（即超过一定的处理时限）， 那么这个客户端会把这些超时消息重新加入到队列中。<br />

**循环列表**
&ensp;&ensp;RPOPLPUSH 命令的 source 和 destination 是相同的话， 那么客户端在访问一个拥有n个元素的列表时，可以在 O(N) 时间里一个接一个获取列表元素， 而不用像 LRANGE 那样需要把整个列表从服务器端传送到客户端。<br />
&ensp;&ensp;上面这种模式即使在以下两种情况下照样能很好地工作： 
* 有多个客户端同时对同一个列表进行旋转（rotating）：它们会取得不同的元素，直到列表里所有元素都被访问过，又从头开始这个操作。 
* 有其他客户端在往列表末端加入新的元素。<br />
&ensp;&ensp;这个模式让我们可以很容易地实现这样一个系统：有 N 个客户端，需要连续不断地对一批元素进行处理，而且处理的过程必须尽可能地快。 一个典型的例子就是服务器上的监控程序：它们需要在尽可能短的时间内，并行地检查一批网站，确保它们的可访问性。<br />
&ensp;&ensp;值得注意的是，使用这个模式的客户端是易于扩展（scalable）且安全的（reliable），因为即使客户端把接收到的消息丢失了， 这个消息依然存在于队列中，等下次迭代到它的时候，由其他客户端进行处理。<br />

BRPOPLPUSH source destination timeout<br />
&ensp;&ensp;弹出一个列表的值，将它推到另一个列表，并返回它;或阻塞，直到有一个可用，RPOPLPUSH的阻塞版本。timeout的单位是秒，当timeout为0的时候，表示无限期阻塞。<br />


*延伸问题：假如主从同步的两个机器都在同一个机房，突发断电，如何在 Redis 宕机的时候保护现场*

参考链接：
1. [Redis RPOPLPUSH 文档](http://www.redis.cn/commands/rpoplpush.html "Redis RPOPLPUSH 文档")

---
layout:     post
title:      PHP,Swoole,Redis list实现简单消息推送
subtitle:   PHP,Swoole,Redis list实现简单消息推送
date:       2019-02-16
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - PHP
    - Redis
    - Swoole
    - Redis List
    - 消息推送
---
**主要是利用 Swoole 的 Redis 协程属性及 Redis list 的阻塞读，当 list 有新消息时， swoole redis 能获取到消息，推送给链接的 fd**

*swoole.html*
用户端代码没有变，还是原来的聊天室页面，具体参见上篇文章

*swoole.php*
```php
<?php
$server = new swoole_websocket_server("0.0.0.0", 9502);

$server->on('workerStart', function ($server, $workerId) {
    $redis = new Swoole\Coroutine\Redis();
    $redis->connect('127.0.0.1', 6379);
    while (true) {
        // brpop 第二个参数 50 表示超时（阻塞等待）时间, blpop 同理，详情建议读文档,对应的 redis 操作是 rpush/lpush key content

        if (($message = $redis->brpop('message', 50)) === null) {
            continue;
        }
        // var_dump($message);  结果为数组

        foreach ($server->connections as $fd) {
            $server->push($fd, 'redis 的 ' . $message[0] . ' 队列发送消息:' . $message[1]);
        }
    }
});

$server->on('open', function ($server, $request) {
    $server->push($request->fd, "hello;\n");
});

$server->on('message', function (swoole_websocket_server $server, $request) {
    $server->push($request->fd, "hello");
});

$server->on('close', function ($server, $fd) {
    echo "client-{$fd} is closed\n";
    $server->close($fd);
});

$server->start();
```

方式如常：`php swoole.php` 启动 `swoole` 进程，然后通过浏览器访问 `http://localhost/swoole.html` , 新起终端，输入 `redis-cli` 进入 redis, 然后执行： `rpush message xxx`, 往 message list 中写入一个数据，swoole 监听着代码，然后就会自动打印数据并推送到 fd，效果如图：

![swoole redis list 效果](https://alpha2016.github.io/img/2019-02-18-swoole-redis-list-demo.jpg "swoole redis list 效果")

> 进阶思考：以上的示例为向所有在线的 fd 推消息，如果一个用户登录之后，在 redis 的 hash 中保存用户 id 对应的 fd，而 list 的名称为 user_1_messages, 这样的名称，中间为用户具体 id, 则用户可以消费自己的队列数据，通知在 hash 中查找用户 id 对应的 fd 值，向对应的 fd 推消息。这样就算是**一个较完整的简单消息消费系统了**，有时间在写具体代码。

**tips: 一次性推送多条数据，例如 `rpush message content1 content2 content3`, brpop 的顺序为 `content3 content2 content1`,则推送顺序也为 3 2 1**
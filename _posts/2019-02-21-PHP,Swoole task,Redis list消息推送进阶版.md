---
layout:     post
title:      PHP,Swoole Task,Redis list消息推送进阶版
subtitle:   PHP,Swoole Task,Redis list消息推送进阶版
date:       2019-02-21
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - PHP
    - Redis
    - Swoole task
    - Redis List
    - 消息推送
---
> 2019.03.15 修正，请参见 [新文章](https://alpha2016.github.io/2019/03/15/Swoole-task%E8%AF%B4%E6%98%8E%E5%8F%8A%E7%BB%93%E5%90%88Redis%E8%BF%9E%E6%8E%A5%E6%B1%A0%E5%AE%9E%E7%8E%B0%E6%B6%88%E6%81%AF%E6%8E%A8%E9%80%81/)

> 测试swoole版本为 4.2.13，旧版本 swoole 可能不兼容

> 应用场景：论坛系统中，给用户推送自己文章的点赞，评论等信息，电商系统中推送购物车商品优惠信息，物流信息等

*起源于设想的两个针对用户消息推送的方案*
> 方案1: 用户登录之后，通过 Websocket 链接 Swoole 服务，然后 Swoole 根据 userid 信息，去 redis 中查找名为 user_id_messages 的队列，获取数据，推送给当前用户的 fd，完成消息的消费。优势在于 Swoole 只负责获取和消费数据，业务端产生的数据扔到对应的队列就可以的。

> 方案2: 用 redis set 来记录用户登录之后和 fd 的绑定值，同时基于此判断用户是否在线，如果不在线，则直接在 list 中保存数据，用户上线时用 Swoole 去获取自己队列中的信息，完成推送，而用户在线时，业务端产生的数据，直接推送。

**个人采用第一种方法，这样在一个完善的系统，业务产生的数据直接扔到 redis list中，而通知则是 swoole 专门负责，对此来说功能分明，对于监控和报警业务异常比较友好，排查和重启服务也比较友好**

效果如下图：

![给指定用户推送消息效果图](https://alpha2016.github.io/img/2019-02-21-php-swoole-task-redis-demo.jpg "给指定用户推送消息demo")


用户端代码  `swoole.html`
**只是小demo，直接明文传user_id参数，正式项目建议加密或者使用token**

```html
<!DOCTYPE html>
<html>
<head>
      <title>swoole chat room</title>
      <meta charset="UTF-8">
      <script type="text/javascript">
          if(window.WebSocket){
              var webSocket = new WebSocket("ws://127.0.0.1:9502?user_id=" + parseInt(Math.random()*1000,10)+1);
              webSocket.onopen = function (event) {
                  //webSocket.send("Hello,WebSocket!"); 
              };
              webSocket.onmessage = function (event) {
                var content = document.getElementById('content');
                content.innerHTML = content.innerHTML.concat('<p style="margin-left:20px;height:20px;line-height:20px;">'+event.data+'</p>');
              }
              
              var sendMessage = function(){
                  var data = document.getElementById('message').value;
                  webSocket.send(data);
              }
          }else{
              console.log("您的浏览器不支持WebSocket");
          }
      </script>
</head>
<body>
    <div style="width:600px;margin:0 auto;border:1px solid #ccc;">
        <div id="content" style="overflow-y:auto;height:300px;"></div>
        <hr/>
        <div style="height:40px">
            <input type="text" id="message" style="margin-left:10px;height:25px;width:450px;">
            <button onclick="sendMessage()" style="height:28px;width:75px;">发送</button>
        </div>
    </div>
</body>
</html>
```

业务代码：`swoole.php`

```php
<?php
$server = new swoole_websocket_server("0.0.0.0", 9502);

// swoole 4.2.13 版本，在 task 中使用协程，需要增加配置项： task_enable_coroutine => true

$server->set(
    [
        'task_worker_num' => 1,
        'task_enable_coroutine' => true
    ]
);

$server->on('open', function ($server, $request) {
    $userId = $request->get['user_id'];
    $server->task(json_encode(['fd' => $request->fd, 'user_id' => $userId]));
    $server->push($request->fd, "hello;\n");
});

// swoole 4.2.x 版本之后的 task 调用闭包函数参数为 2 个，旧版本为 4个

$server->on('task', function ($server, Swoole\Server\Task $task) {
    $redis = new Swoole\Coroutine\Redis();
    $redis->connect('127.0.0.1', 6379);
    $data = json_decode($task->data, true);
    echo 'fd=' . $data['fd'] . ',,,user_id=' . $data['user_id'];
    $list = 'user_' . $data['user_id'] . '_messages';
    while (true) {
        // brpop 第二个参数 50 表示超时（阻塞等待）时间, blpop 同理，详情建议读文档,对应的 redis 操作是 rpush/lpush key content
        
        if (($message = $redis->brpop($list, 50)) === null) {
            continue;
        }
        // var_dump($message); // 结果为数组
        
        $server->push($data['fd'], 'redis 的 ' . $message[0] . ' 队列发送消息:' . $message[1]);
    }
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

参考链接：
1. [swoole task文档](https://wiki.swoole.com/wiki/page/54.html "swoole task 文档")
2. [memory 文章](https://www.im050.com/posts/380 "memory swoole task 文章")

**期间 memory 博客作者及其他大佬提供了思路，才得以完成，特此感谢，©原创文章**

---
layout:     post
title:      PHP,Swoole,Redis PubSub实现简单消息推送
subtitle:   PHP,Swoole,Redis PubSub实现简单消息推送
date:       2019-02-16
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - PHP
    - Redis
    - Swoole
    - Redis PubSub
    - 消息推送
---

> 这个是最简单的初级版本

应用场景：**限于不很严谨的消息推送系统，主要是给在线用户推送消息**
- 类似论坛新加精/置顶一篇文章，需要通知一下在线的用户
- 活动已经通知过全部用户，在开始前通知一下在线用户的场景

&ensp;&ensp;前提先确保安装并启动了 nginx/apache,php,redis服务，php的swoole扩展已经安装才可以，这些具体安装和配置方法请自行搜索。执行一下 `php -m` 后可以看到swoole扩展，就是安装成功。

&ensp;&ensp;个人的测试系统是Ubuntu 18.04 LTS，带用户界面，因为需要浏览器进行测试。

![服务及端口](https://alpha2016.github.io/img/2019-02-15-php-swoole-redis-network.jpg "当前服务及端口")

在网站目录，默认的nginx配置的目录为/var/www/html,个人自定义为了/var/www 目录，直接建立 swoole.html ,  swoole.php

*swoole.html*
```html
<!DOCTYPE html>
<html>
<head>
      <title>swoole chat room</title>
      <meta charset="UTF-8">
      <script type="text/javascript">
          if(window.WebSocket){
              // 端口和ip地址对应不要写错
              var webSocket = new WebSocket("ws://127.0.0.1:9502");
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

*swoole.php*
```php
$server = new swoole_websocket_server("0.0.0.0", 9502);

$server->on('workerStart', function ($server, $workerId) {
    $client = new swoole_redis;
    $client->on('message', function (swoole_redis $client, $result) use ($server) {
        if ($result[0] == 'message') {
            var_dump($result);
            foreach($server->connections as $fd) {
                $server->push($fd, '频道为： ' $result[1] . ' 发送消息：' . $result[2]);
            }
        }
    });
    $client->connect('127.0.0.1', 6379, function (swoole_redis $client, $result) {
        $client->subscribe('message');   // 队列名称可以自定义
    });
});

$server->on('open', function ($server, $request) {

});

// 屏蔽这段会报错
$server->on('message', function (swoole_websocket_server $server, $frame) {
    $server->push($frame->fd, "hello");
});

$server->on('close', function ($serv, $fd) {

});

$server->start();
```

在 `/var/www` 目录下，执行 `php swoole.php` 启动 swoole 进程，然后通过浏览器访问 `localhost://swoole.html` ，就可以看到聊天室页面，新起终端，使用 `redis-cli` 命令进入redis, 可以通过 `pubsub channels` 命令查看当前的频道，就可以看到 message 频道，然后 `publish message 消息体`, 在 message 频道发布一条消息，在聊天室页面就可以看到消息，效果如同：
![swoole redis pubsub 效果](https://alpha2016.github.io/img/2019-02-15-php-swoole-redis-demo.jpg "swoole redis pubsub 效果")

> 这个有一些缺憾是：redis pubsub 自身的缺憾，消息不能持久化，重启redis消息都会丢失。swoole 创建的频道在 swoole 服务退出的时候，频道也会关闭，消息丢失。所以适合搭一个简单给所有在线的用户推消息的体系，对消息必达的要求不高。

**todo 更完善严谨的消息推送系统**

参考链接：
1. [我的sf文章](https://segmentfault.com/a/1190000008908533) 
2. [Redis订阅消息转发到 Webscoket](https://segmentfault.com/a/1190000010986855)
3. [Swoole 速查表](https://toxmc.github.io/swoole-cs.github.io/)   
 
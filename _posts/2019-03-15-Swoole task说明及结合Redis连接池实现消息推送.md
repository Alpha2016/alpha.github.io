---
layout:     post
title:      Swoole task详解及结合Redis连接池实现消息推送
subtitle:   Swoole task详解及结合Redis连接池实现消息推送
date:       2019-03-15
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - PHP
    - Redis
    - Swoole task
    - Redis 连接池
    - 消息推送
---

> 起源于上一篇利用 Swoole task 实现消息推送，被打脸，重新去学习了一下 Swoole 的 task,他不适合处理长时间阻塞读的场景，高并发了更会搞笑了，[原文链接](https://alpha2016.github.io/2019/02/21/PHP,Swoole-task,Redis-list%E6%B6%88%E6%81%AF%E6%8E%A8%E9%80%81%E8%BF%9B%E9%98%B6%E7%89%88/),不考虑场景乱写太xx

task 文档中有此说明：<br />
未指定目标 Task 进程，调用 task 方法会判断 Task 进程的忙闲状态，底层只会向处于空闲状态的 Task 进程投递任务。如果所有 Task 进程均处于忙的状态，底层会轮询投递任务到各个进程。可以使用 server->stats 方法获取当前正在排队的任务数量。<br />
task 操作的次数必须小于 onTask 处理速度，如果投递容量超过处理能力，task会塞满缓存区，导致worker进程发生阻塞。worker 进程将无法接收新的请求。<br />

这两个意味着用 task 去 brpop redis list，会阻塞 task，新的任务只能被动等待处理，这相当于把同时处理的任务压制到很低了，基本不可用。task 处理较耗时但并发小的问题还算不错，他的实现是 `task底层使用Unix Socket管道通信，是全内存的，没有IO消耗。单进程读写性能可达100万/s，不同的进程使用不同的管道通信，可以最大化利用多核`，性能非常高。

额外说明：<br />
**缓存区中的Task数据，在重启进程后会丢失吗？？**<br />
不会丢失，但是不一定会去读取，取决于 `task_ipc_mode`

task_ipc_mode,  默认值为1 <br />
message_queue_key, 设置消息队列的KEY，仅在`task_ipc_mode = 2/3`时使用

默认`task_ipc_mode`模式下（等于1），重启 master 进程，未消费的队列不会再消费，但是会存储在缓冲区，当为2/3模式时，自然会去读取未消费的队列（要相同的 key），一个 message_queue_key 意味着不同的队列，同一台机器多开队列消费时应该注意设置不同的 key <br />

**原来的针对指定用户进行消息推送改写成以下的代码，swoole.php 部分使用 redis 连接池，相当于多个协程使用一个协程客户端，减少链接数量和过程耗时，直接在 open 事件中 brpop redis list，不用额外新起 worker 去处理任务，以下是代码：**

用户端代码  `swoole.html`
**只是小demo，直接明文传user_id参数，正式项目建议加密或者使用token** <br />
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

业务代码：`swoole.php` <br />
```php
<?php
$server = new swoole_websocket_server("0.0.0.0", 9502); 
$pool = new RedisPool();

$server->set([
    // 如开启异步安全重启, 需要在workerExit释放连接池资源

    'reload_async' => true
]);

$server->on('start', function (swoole_http_server $server) {
    var_dump($server->master_pid);
});

// 退出事件

$server->on('workerExit', function (swoole_http_server $server) use ($pool) {
    $pool->destruct();
});

$server->on('open', function ($server, $request) use ($pool) {
    $userId = $request->get['user_id'];
    // redis 连接池
    
    $redis = $pool->get();
    
    if ($redis === false) {
        $server->push($request->fd, "redis error;\n");
        return;
    }

    $list = 'user_' . $userId . '_messages';
    echo $list;
    while (true) {
        // brpop 第二个参数 50 表示超时（阻塞等待）时间, blpop 同理，详情建议读文档,对应的 redis 操作是 rpush/lpush key content
        
        if (($message = $redis->brpop($list, 50)) === null) {
            continue;
        }
        // var_dump($message); // 结果为数组
        
        $server->push($request->fd, 'redis 的 ' . $message[0] . ' 队列发送消息:' . $message[1]);
    }

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

// 连接池代码

class RedisPool
{
    protected $available = true;
    protected $pool;

    public function __construct()
    {
        $this->pool = new SplQueue;
    }

    public function put($redis)
    {
        $this->pool->push($redis);
    }

    /**
     * @return bool|mixed|\Swoole\Coroutine\Redis
     */
    public function get()
    {
        //有空闲连接且连接池处于可用状态
        
        if ($this->available && count($this->pool) > 0) {
            return $this->pool->pop();
        }

        //无空闲连接，创建新连接
        
        $redis = new Swoole\Coroutine\Redis();
        $res = $redis->connect('127.0.0.1', 6379);
        if ($res == false) {
            return false;
        } else {
            return $redis;
        }
    }

    public function destruct()
    {
        // 连接池销毁, 置不可用状态, 防止新的客户端进入常驻连接池, 导致服务器无法平滑退出
        
        $this->available = false;
        while (!$this->pool->isEmpty()) {
            $this->pool->pop();
        }
    }
}

```

效果图：<br />
![Swoole Redis Pool 实现消息推送](https://alpha2016.github.io/img/2019-03-15-swoole-redis-pool-demo.jpg "Swoole Redis Pool 实现消息推送")

参考链接：
1. [Swoole task 文档](https://wiki.swoole.com/wiki/page/134.html "Swoole task文档")
2. [Swoole 在多个协程间使用协程客户端](https://wiki.swoole.com/wiki/page/852.html "Swoole 在多个协程间使用协程客户端")
3. [task_ipc_mode 文档](https://wiki.swoole.com/wiki/page/296.html "task_ipc_mode 文档")


最后恰饭 [阿里云全系列产品/短信包特惠购买 中小企业上云最佳选择 阿里云内部优惠券](https://www.aliyun.com/minisite/goods?userCode=0amqgcs9)
---
layout:     post
title:      Swoole结合Vuejs实现动态图表变化
subtitle:   Swoole结合Vuejs实现动态图表变化
date:       2019-02-27
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - PHP
    - Vuejs
    - Swoole
---

> 大概需求是展示类似股票行情折线图的一个页面，借助 vuejs 的动态渲染和 swoole 长链接的特性，不刷新页面 + 不 ajax 轮询去请求数据，差不多符合新技术了，用户端主要用到了 echarts 和 vuejs

&ensp;&ensp;逻辑就是打开页面的时候，自行读取以往的数据，同时链接到 swoole 服务，swoole 中 brpop 负责监听 redis 的数据更新队列，有新数据的时候，所有当前用户推送， js 在用户段控制，将新数据 push 到现有数据。

用户端 *swoole-charts.html*

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <title>v-charts swoole 动态变更</title>
    <meta charset="UTF-8">
</head>

<body>
    <div id="app">
        <ve-line :data="chartData"></ve-line>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/echarts/dist/echarts.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/v-charts/lib/line.min.js"></script>
    <!-- -------------------------------------------------△△△△------------ -->
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/v-charts/lib/style.min.css">
    <script>
        var vm = new Vue({
            el: '#app',
            data: {chartData: {
                columns: ['日期', '销售额'],
                rows: [
                    { '日期': '1月1日', '销售额': 123 },
                    { '日期': '1月2日', '销售额': 1223 },
                    { '日期': '1月3日', '销售额': 2123 },
                    { '日期': '1月4日', '销售额': 4123 },
                    { '日期': '1月5日', '销售额': 3123 },
                    { '日期': '1月6日', '销售额': 7123 }
                ]
            }},
            components: { VeLine }
        });

        if (window.WebSocket) {
            var webSocket = new WebSocket("ws://127.0.0.1:9502");
            webSocket.onopen = function (event) {
                // webSocket.send("Hello,WebSocket!"); 
            };

            // 新消息来的时候, vue 自身的语法，参考：https://cn.vuejs.org/v2/guide/list.html#%E5%8F%98%E5%BC%82%E6%96%B9%E6%B3%95 

            webSocket.onmessage = function (event) {
                console.log(JSON.parse(event.data));
                vm.chartData.rows.push(JSON.parse(event.data));
            }
        } else {
            console.log("您的浏览器不支持WebSocket");
        }
    </script>
</body>
</html>

```

业务端：*swoole-charts.php*

```php
<?php
$server = new swoole_websocket_server("0.0.0.0", 9502);

$server->on('workerStart', function ($server, $workerId) {
    $redis = new Swoole\Coroutine\Redis();
    $redis->connect('127.0.0.1', 6379);
    while (true) {
        // brpop 第二个参数 50 表示超时（阻塞等待）时间, blpop 同理，详情建议读文档,对应的 redis 操作是 rpush/lpush key content 
        
        if (($message = $redis->brpop('data', 50)) === null) {
            continue;
        }
        // var_dump($message); 结果为数组 
        
        foreach ($server->connections as $fd) {
            echo $message[1] . '--' . gettype($message[1]) . PHP_EOL;
            $server->push($fd, $message[1]);
        }
    }
});

$server->on('open', function ($server, $request) {
    // nothing
    
});

$server->on('message', function (swoole_websocket_server $server, $request) {
    // nothing
    
});

$server->on('close', function ($server, $fd) {
    echo "client-{$fd} is closed\n";
    $server->close($fd);
});

$server->start();

```

效果如图：

![Swoole结合Vuejs 实现动态图表变化](https://alpha2016.github.io/img/2019-02-27-swoole-charts-demo.jpg "vue swoole 动态图表")

执行过程如常：`php swoole-charts.php` 启动服务，开始监听 redis 的 data 队列，然后用户访问到 `http://localhost/swoole-charts.html` 页面，最开始加载在页面中的数据，然后新起终端，输入 `redis-cli` 打开 redis，`rpush data '{"日期":"1月7日","销售额":358}'` ... 等，新增的数据推送到 data 队列，然后 swoole 负责读取和推送，vuejs 负责根据数据变化，动态渲染图表

**©原创文章**

最后恰饭 [阿里云全系列产品/短信包特惠购买 中小企业上云最佳选择 阿里云内部优惠券](https://www.aliyun.com/minisite/goods?userCode=0amqgcs9)
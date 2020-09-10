---
layout:     post
title:      Swoole 结合jwt auth实现登录聊天功能
subtitle:   Swoole 结合jwt auth实现登录聊天功能
date:       2019-04-14
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - Swoole
    - jwt auth
    - PHP
    - 聊天室demo
---

> 整体逻辑：在聊天室页面，前期发送消息按钮不能点击，需要先登录，然后用户登录之后，在 login.php 方法验证用户信息，并使用 jwt 生成 token 并返回，前端收到 token 之后，可以选择保存到cookie中，也可以只在此页面短时间使用，在建立 websocket 链接的时候，带过去 token 信息，swoole 在 open 事件中收到 token, 解析 token, 如果验证失败则断开链接，成功则可以聊天。

个人写的 demo 在 swoole 文件夹下，在此文件夹下执行 `composer require firebase/php-jwt` ,安装 jwt 包，使用具体可以参考包文档。其他所需是 swoole.html, login.php, swoole.php 三个前后端页面，代码如下：

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

            var login = function () {
                var name = document.getElementById('name').value;
                var data = new FormData();
                data.append('name', name);

                var request = new XMLHttpRequest();
                request.open("post", 'login.php');
                request.send(data);
                request.onload = function() {
                    if(request.readyState === 4 && request.status == 200) {
                        var webSocket = new WebSocket("ws://127.0.0.1:9502?token=" + request.responseText);
                        webSocket.onopen = function (event) {
                            //webSocket.send("Hello,WebSocket!");
                        };
                        webSocket.onmessage = function (event) {
                            var content = document.getElementById('content');
                            content.innerHTML = content.innerHTML.concat('<p style="margin-left:20px;height:20px;line-height:20px;">'+event.data+'</p>');
                        };

                        // 处理发送事件
                        
                        var sendButton = document.getElementById('send');
                        sendButton.disabled = false;
                        sendButton.innerHTML  = '发送';
                        sendButton.onclick = function(){
                            var data = document.getElementById('message').value;
                            webSocket.send(data);
                        };
                    } else {
                        alert('error');
                    }
                };

            }
        }else{
            console.log("您的浏览器不支持WebSocket");
        }
    </script>
</head>
<body>
<div style="width:600px;margin:0 auto;border:1px solid #ccc;">
    <div style="height:40px">
        <input type="text" id="name" style="margin-left:10px;height:25px;width:450px;">
        <button onclick="login()" style="height:28px;width:75px;">登录</button>
    </div>
    <hr />
    <div id="content" style="overflow-y:auto;height:300px;"></div>
    <hr/>
    <div style="height:40px">
        <input type="text" id="message" style="margin-left:10px;height:25px;width:450px;">
        <button id="send" style="height:28px;width:75px;" disabled="disabled">登录后可以发送</button>
    </div>
</div>
</body>
</html>
```

*login.php*
```php
<?php
require __DIR__.'/vendor/autoload.php';
use Firebase\JWT\JWT;

// 前提检查用户信息，验证通过则返回数据，demo 跳过这步
// 文档地址： https://packagist.org/packages/firebase/php-jwt

$key = "swoole";
$token = array(
    "iss" => "localhost",
    "aud" => "localhost",
    "iat" => time(),
    "nbf" => time()  + 3600,
    "name" => $_POST['name']
);


$jwt = JWT::encode($token, $key);
echo $jwt;
```

*swoole.php*
```php
<?php
require 'vendor/autoload.php';
use Firebase\JWT\JWT;
$server = new swoole_websocket_server("0.0.0.0", 9502);

$server->on('open', function ($server, $request) {
    $key = 'swoole';
    $token = $request->get['token'] ?? '';

    $decode = JWT::decode($token, $key, ['HS256']);
    if (!$decode) {
        $server->close($request->fd);
        return false;
    }

    $decodeArray = (array)$decode;
    $server->push($request->fd, "hello;" . $decodeArray['name'] . "\n");
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

© 原创文章，如遇到问题可以联系邮箱：hexiaodong1992@outlook.com


最后恰饭 [阿里云全系列产品/短信包特惠购买 中小企业上云最佳选择 阿里云内部优惠券](https://www.aliyun.com/minisite/goods?userCode=0amqgcs9)

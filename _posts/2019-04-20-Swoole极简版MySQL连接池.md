---
layout:     post
title:      Swoole 极简版MySQL连接池
subtitle:   Swoole 极简版MySQL连接池
date:       2019-04-17
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - Swoole
    - MySQL连接池
---

> 额外注意：MySQL 连接池只能减轻业务连接数据库时候的操作，他允许同一个连接多次被使用，而不是每次重新创建连接。需要额外注意的最小连接数和最大连接数，
最小连接数意味着一直需要维护的资源，如果不需要这么多连接，将造成资源浪费。而最大连接数则是超出这个数量，后面的连接请求将需要等待，也会有一些影响。最大连接数也受 `max_connections` 配置数量的制约。

*mysqlPool.php*
```php
<?php
use Swoole\Runtime;

Runtime::enableCoroutine(true);

class MysqlPool
{
    protected $min;
    protected $max;
    protected $queue;

    protected $config = [
        'host'     => '127.0.0.1',
        'port'     => 3306,
        'username' => 'root',
        'password' => 'root',
        'db'       => 'business',
        'charset'  => 'utf8'
    ];


    public function __construct($min = 10, $max = 100)
    {
        $this->min = $min;
        $this->max = $max;
        $this->queue = new SplQueue();
    }


    /**
     * 初始化
     */
    public function init()
    {
        for ($i = 0; $i < $this->min; $i++) {
            $this->generate();
        }
    }


    /**
     * 生成连接
     */
    public function generate()
    {
        $swooleMysql = new Swoole\Coroutine\MySQL();
        $swooleMysql->connect([
            'host'     => $this->config['host'],
            'port'     => $this->config['port'],
            'user'     => $this->config['username'],
            'password' => $this->config['password'],
            'database' => $this->config['db'],
            'charset'  => $this->config['charset']
        ]);

        $this->queue->push($swooleMysql);
    }


    /**
     * @param integer $timeout
     * 获取连接
     */
    public function getConnection($timeout = 10)
    {
        $time = time();
        while ($time + $timeout > time()) {
            try {
                $connection = $this->queue->pop();
                break;
            } catch (Exception $e) {
                sleep(1);
            }
        }

        return $connection;
    }


    /**
     * @param string $connection
     * 释放连接
     */
    public function free($connection)
    {
        $this->queue->push($connection);
    }


    /**
     * 维持当前的连接数不断线，并且剔除断线的链接 .
     */
    public function keepAlive()
    {
        swoole_timer_tick(2000, function() {
            while ($this->queue->count() > 0 && $next = $this->queue->shift()) {
                $next->query("select 1" , function($db, $res){
                    
                    if ($res === false) {
                        return;
                    }
                    
                    echo "当前连接数：" . $this->queue->count() . PHP_EOL;

                    $this->queue->push($db);
                });
            }
        });

        swoole_timer_tick(2000 , function() {
            if($this->queue->count() > $this->max) {
                while($this->max < $this->queue->count()) {
                    $next = $this->queue->shift();
                    $next->close();
                    
                    echo "关闭连接..." . PHP_EOL;
                }
            }
        });
    }
}

// 测试部分

go(function () {
    $pool = new MysqlPool();
    $pool->init();
    
    $num = 200;
    while ($num-- > 0) {
        go(function () use ($pool) {
            $conn = $pool->getConnection();
            if (!$conn) {
                echo "get conn error.\n";
                return;
            }
            $statement = $conn->prepare('select * from users where id = ?');
            $id = mt_rand(0, 10);
            $result = $statement->execute([$id]);
            var_dump($result);
            $pool->free($conn);
        });
    }
});

```

取消注释的测试部分代码，然后执行 `php mysqlPool.php` 进行测试，连接池默认的最大连接是 100，测试中是 200 个连接，超出部分将有一个等待过程，为了方便查看数据，可以将这两个数字都调小。

keepAlive 方法可以在 workerStart 事件中调用，直接利用 swoole 的定时器特性轮训，维护连接池状态。

© 原创文章

其他参考：[韩天峰 MySQL连接池](http://rango.swoole.com/archives/265)

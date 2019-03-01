---
layout:     post
title:      PHP 及 Swoole 读取大文本文件的几种方式
subtitle:   PHP 及 Swoole 读取大文本文件的几种方式
date:       2019-03-01
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - PHP
    - Swoole
    - 读取大文件
---

> 这种问题主要出现在面试题里，现实版没见过，如果有的话，可能是导入 excel 数据的时候用到，一次导入 100m+ 大小的一个 excel 文件，php 配置的 memory_limit 只有10m，如果操作

**一般题目为，机器内存为2g，然后有一个 10g 的大文件，请问如何利用 php 读取，基于此需求，我导出的 sql 文件为4m ，然后几次复制粘贴，就 222m 了，然后将 php memory_limit 设置成 16m,重启服务生效**

基本思维就是一个字符一个字符的读，一行一行的读，或者一块一块的读，实现也是这样的，代码如下

*单字符的读*

```php
<?php
$begin = microtime( true );
$fp = fopen( 'aaa.sql', "r" );
while( false !== ( $ch = fgetc( $fp ) ) ){
  // 打开注释后屏显字符会严重拖慢程序速度！也就是说程序运行速度可能远远超出屏幕显示速度
  // echo $char.PHP_EOL;
  
}
fclose( $fp );
$end = microtime( true );
echo "cost : ".( $end - $begin ).PHP_EOL;

// 结果 22s+
```

*单行读*

```php
<?php
$begin = microtime( true );
$fp = fopen( 'aaa.sql', 'r' );
while( false !== ( $buffer = fgets( $fp, 4096 ) ) ){
  // echo $buffer.PHP_EOL;
  
}
if( !feof( $fp ) ){
  throw new Exception('... ...');
}
fclose( $fp );
$end = microtime( true );
echo "cost : ".( $end - $begin ).' sec'.PHP_EOL;

// 0.314s + 
```

*分块读*

```php
<?php
$begin = microtime( true );
$fp = fopen( 'aaa.sql', 'r' );
while( !feof( $fp ) ){
  // 如果你要使用echo，那么，你会很惨烈...
  fread( $fp, 10240 );
}
fclose( $fp );
$end = microtime( true );
echo "cost : ".( $end - $begin ).' sec'.PHP_EOL;
exit;

// 0.2s+
```

*Swoole 异步读取方法*

```php
<?php
use Swoole\Async;

$trunk_size = 102400;
$offset = 0;
$begin = microtime( true );

swoole_async_read("aaa.sql", function($fileName, $content) use ($begin) {
    // var_dump($fileName, strlen($content));
    $end = microtime( true );
    echo "cost : " . ( $end - $begin ).' sec'.PHP_EOL;
    return true;
}, $trunk_size, $offset);

// 2s+
```

最终对比，分块读效率最高，能够自定义每次读取块的大小，在 memory_limit 之下，就不会产生问题，swoole 的异步读取底层应该也是分块读吧，没去看源码了，这几种方式都能实现有限内存内读取大文件，自行根据业务需求选择就行。**其他还有利用 yield 协程读取，这样读取时间不如分块读，但内存占用会很小**

参考链接：
1. [elarilty 文章](https://segmentfault.com/a/1190000017205171 "elarilty 文章")
2. [Swoole 文档](https://wiki.swoole.com/wiki/page/188.html "Swoole 文档")

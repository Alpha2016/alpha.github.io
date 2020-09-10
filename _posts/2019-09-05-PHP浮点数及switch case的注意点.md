---
layout:     post
title:      PHP浮点数及switch case的注意点
subtitle:   PHP浮点数及switch case的注意点
date:       2019-09-05
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - PHP
    - 浮点数
    - switch case
    - Ieee754
---

##### 浮点数的注意点

浮点数基本是各个语言的老生常谈了，尤其 PHP 这种弱类型的语言，最常见的坑是电商活动的时候，偶尔设置价格是 19.9 元，调用微信支付，转换为分，就是 `intval(19.90 * 100)` 是 1989 分，对账的时候很容易踩坑，生成价格或者腾讯传回来的，和本地的 decimal 类型数据 * 100 就不一致了，可以使用 [bcmul](https://www.php.net/manual/zh/function.bcmul.php) 函数避免错误，`bcmul(19.90, 100, 0)` 这样的，会直接返回 1990， bcmul 是专门处理大精度数据之间的乘法运算，**在 PHP7.3.0 及以上需要专门需要注意保留两位小数时，遇到最后为 .00 的情况**。

浮点数入门级理解可以参考 [PHP 浮点数的一个常见问题的解答](http://www.laruence.com/2013/03/26/2884.html)、[关于PHP浮点数你应该知道的(All 'bogus' about the float in PHP)](http://www.laruence.com/2011/12/19/2399.html)，至于为什么这么定义，就需要了解 [Ieee 754 协定](https://zh.wikipedia.org/wiki/IEEE_754)，维基百科上说的很明确了，从这个的由来到采用，面试中能答出来原因，PHP 的实现及为什么，这就非常优秀了。

##### switch case
起源是一个群友问的问题，代码如下：
```php
<?php
$a = 0;
switch($a){
    case $a >= 0:
        echo 1;
        break;
    case $a >= 10:
        echo 2;
        break;
    default:
        echo 3;
        break;
} 
# 输出2
```
**因为 case 之后不能跟运算符，PHP 会优先处理这些，将他们计算得到常量，然后进行判断**，转换后的代码大概是：
```php
<?php
$a = 0;
switch($a){
    case true:    // $a >= 0:
        echo 1;
        break;
    case false:   // $a >= 10:
        echo 2;
        break;
    default:
        echo 3;
        break;
}
```
弱类型的特性，0 == false, 则最终结果为 2；

© 原创文章


最后恰饭 [阿里云全系列产品/短信包特惠购买 中小企业上云最佳选择 阿里云内部优惠券](https://www.aliyun.com/minisite/goods?userCode=0amqgcs9)
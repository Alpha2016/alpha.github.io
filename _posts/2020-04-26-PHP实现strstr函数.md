---
layout:     post
title:      PHP 实现 strstr 函数
subtitle:   PHP 实现 strstr 函数
date:       2020-04-26
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - Go
    - PHP
    - LeetCode
    - strstr
---


**PHP 实现**：
```php
function strstr2($str, $search) {
    $count = strlen($str);
    $tmpLength = 0;
    $tmp = [];
    $i = 0;
    while ($i < $count) {
        // 未找到置为初始值
        if ($str[$i] !== $search[$tmpLength]) {
            $tmp = [];
            $tmpLength = 0;
            if ($i === $count - 1) {
                return -1;
            }
        } else {
            // 找到匹配元素，继续执行
            $tmp[$tmpLength] = $str[$i];
            if (!isset($search[$tmpLength + 1])) {
                return $i - $tmpLength;  // 位置
                return $tmp;             // 结果集
            }
            $tmpLength++;
        }
        $i++;
    }
}
echo substr2('hello', 'll'); // output: 2
```

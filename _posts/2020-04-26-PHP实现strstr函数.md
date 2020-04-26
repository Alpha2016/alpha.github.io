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
public function strStr(string $haystack, string $needle): int
    {
        if (0 === ($lengthOfNeedle = strlen($needle))) {
            return 0;
        }

        $lengthOfHaystack = strlen($haystack);
        $diffLength = $lengthOfHaystack - $lengthOfNeedle;

        for ($i = 0, $j = 0, $k = 0; $i <= $diffLength; $i++, $j = 0, $k = $i) {
            while ($k < $lengthOfHaystack && $j < $lengthOfNeedle && $haystack[$k] === $needle[$j]) {
                $k++;
                $j++;
            }

            if ($j === $lengthOfNeedle) {
                return $k - $j;
            }
        }

        return -1;
    }
echo strStr('hello', 'll'); // output: 2
```

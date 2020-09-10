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


**PHP 实现V2**
```php
function strStr2($haystack, $needle) :int {

    // 子串为空则返回 0
    if ($needle === '') {
        return 0;
    }

    $i = 0;
    $j = 0;
    while ($j < strlen($haystack)) {
        // 判断 haystack[j] 与 needle[i] 是否相等，不想等，则 i 重新指向 needle 字符串的首字符，即 i = 0
        // j 指向 j上一次初始化的后一个字符串，即 j = j - i + 1
        if ($haystack[$j] !== $needle[$i]) {
            $j = $j - $i + 1;
            $i = 0;
        } else {
            // haystack[j] 与 needle[i] 相等，则先判断 i 是否已经最大
            // 是的话就返回 needle 在 haystack 的第一个位置，计算方式： j - i
            if ($i === strlen($needle) - 1) {
                return $j - $i;
            }
            // 如果 i 不是 needle 的最大索引，那么 j 向后移动一位，i 向后移动一位，继续比较
            $j++;
            $i++;
        }
    }
    // j 越界了，说明不存在子串 needle，即返回 -1
    return -1;
}

return strStr2('asissisap', 'is'); // output: 2
```

参考文章： [让我们一起啃算法---- 实现 strStr](https://learnku.com/articles/43524)


最后恰饭 [阿里云全系列产品/短信包特惠购买 中小企业上云最佳选择 阿里云内部优惠券](https://www.aliyun.com/minisite/goods?userCode=0amqgcs9)
---
layout:     post
title:      PHP 实现leetcode打开转盘锁题
subtitle:   PHP 实现leetcode打开转盘锁题
date:       2020-06-30
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - Go
    - PHP
    - LeetCode
    - 打开转盘锁
    - BFS
---

##### 打开转盘锁
你有一个带有四个圆形拨轮的转盘锁。每个拨轮都有10个数字： `'0', '1', '2', '3', '4', '5', '6', '7', '8', '9' `。每个拨轮可以自由旋转：例如把 '9' 变为  '0'，'0' 变为 '9' 。每次旋转都只能旋转一个拨轮的一位数字。

锁的初始数字为 `'0000'`，一个代表四个拨轮的数字的字符串。

列表 `deadends` 包含了一组死亡数字，一旦拨轮的数字和列表里的任何一个元素相同，这个锁将会被永久锁定，无法再被旋转。

字符串 `target` 代表可以解锁的数字，你需要给出最小的旋转次数，如果无论如何不能解锁，返回 -1。

**示例 1:**
```
输入：deadends = ["0201","0101","0102","1212","2002"], target = "0202"
输出：6
解释：
可能的移动序列为 "0000" -> "1000" -> "1100" -> "1200" -> "1201" -> "1202" -> "0202"。
注意 "0000" -> "0001" -> "0002" -> "0102" -> "0202" 这样的序列是不能解锁的，
因为当拨动到 "0102" 时这个锁就会被锁定。
```

**示例 2:**
```
输入: deadends = ["8888"], target = "0009"
输出：1
解释：
把最后一位反向旋转一次即可 "0000" -> "0009"。
```

**示例 3:**
```
输入: deadends = ["8887","8889","8878","8898","8788","8988","7888","9888"], target = "8888"
输出：-1
解释：
无法旋转到目标数字且不被锁定。
```

**示例 4:**
```
输入: deadends = ["0000"], target = "8888"
输出：-1
```

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/open-the-lock

##### 解题思路
[请参见官方题解](https://leetcode-cn.com/problems/open-the-lock/solution/da-kai-zhuan-pan-suo-by-leetcode/)

**php 代码**
```php
// 这道题真的是尽可能的使用高效率函数，plusOne minusOne 方法我额外处理了点就超时
class Solution {

    /**
     * @param String[] $deadends
     * @param String $target
     * @return Integer
     */
    function openLock($deadends, $target) {
        $start = '0000';
        $tried = [];
        $q = new SplQueue();
        $step = 0;
        $q->enqueue($start);
        $tried[$start] = true;

        while (!$q->isEmpty()) {
            $sz = $q->count();
            for ($i = 0; $i < $sz; $i++) {
                $key = $q->dequeue();
                if (in_array($key, $deadends, true)) {
                    continue;
                }

                if ($key === $target) {
                    return $step;
                }

                for ($j = 0; $j < 4; $j++) {
                    $up = plusOne($key, $j);
                    if (!isset($tried[$up])) {
                        $tried[$up] = true;
                        $q->enqueue($up);
                    }

                    $down = minusOne($key, $j);
                    if (!isset($tried[$down])) {
                        $tried[$down] = true;
                        $q->enqueue($down);
                    }
                }
            }

            $step++;
        }

        return -1;
    }
}

function plusOne($s, $j) {
    $s = $s . '';
   if ($s[$j] == '9') {
        $s[$j] = '0';
    } else {
        $s[$j] = (int)$s[$j] + 1;
    }

    return $s;
}

function minusOne($s, $j) {
    $s = $s . '';
    if ($s[$j] == '0') {
        $s[$j] = '9';
    } else {
        $s[$j] = (int)$s[$j] - 1;
    }

    return $s;
}
```


顺带推荐买课,用右侧链接有优惠 [算法面试通关40讲](https://time.geekbang.org/course/intro/130?code=eh3BHyG3lG7AVgwxWXsSgvRJZROaofNh-bg7Fu7lHU4%3D&utm_term=SPoster)
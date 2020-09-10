---
layout:     post
title:      Laravel 手动开关 Eloquent 修改器
subtitle:   Laravel 手动开关 Eloquent 修改器
date:       2019-12-29
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - PHP
    - Laravel
    - Eloquent
    - 修改器
    - Mutators
---

> 测试框架版本是 Laravel 6.5, Eloquent 修改器使用可以参阅 -> [查看文档](https://learnku.com/docs/laravel/6.x/eloquent-mutators/5179)

**修改器的手动开关的场景就是差异化的返回数据，例如在后台管理的时候，图片地址要相对路径，然后 app 端期望返回全路径的地址，这个时候就需要手动开启和关闭了。**

大概操作就是在模型中声明一个静态变量，然后修改器中判断这个静态变量值是 true/false; 如果是 true 则处理，如果为 false 就不处理，具体操作：
```php
    public static $modify = true;

    /**
     * 获取用户的姓名.
     * 判断是否需要修改及 $value 是不是空值
     * @param  string  $value
     * @return string
     */
    public function getFirstNameAttribute($value)
    {
        return self::$modify && $value ? ucfirst($value) : $value;
    }
```

示例代码是默认开启修改器的，无需的话可以关闭修改器，在具体业务层使用前关闭就可以的

```php
User::$modify = false;   // 关闭修改器

return $user:findOrFail(1);
```

如果不手动关闭，想获取原数据，而不是被修改之后的值，也可以这样获取原始值：

```php
$user = User::find(1);

return $user->getOriginal('first_name');
```


最后恰饭 [阿里云全系列产品/短信包特惠购买 中小企业上云最佳选择 阿里云内部优惠券](https://www.aliyun.com/minisite/goods?userCode=0amqgcs9)

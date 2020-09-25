---
layout:     post
title:      MySQL解决 ONLY_FULL_GROUP_BY 错误的几个方法
subtitle:   MySQL解决 ONLY_FULL_GROUP_BY 错误的几个方法
date:       2020-09-25
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - Go
    - PHP
    - MySQL
    - any_value
    - ONLY_FULL_GROUP_BY
---

在 MySQL 5.7版本以上进行一些 `ORDER BY` 或者 `GROUP BY` 时，会出现如下错误 `[Err] 1055 - Expression #1 of ORDER BY clause is not in GROUP BY clause and contains nonaggregated column ‘information_schema.PROFILING.SEQ’ which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by`, MySQL 的约束是：你进行了 `GROUP BY` 和 `ORDER BY`，就需要保证你 `SELECT` 的列都在 `GROUP BY` 和 `ORDER BY` 中，然而实际需求并没有这样简单。

##### 临时修改 ONLY_FULL_GROUP_BY 模式
查询出来现有的模式
```sql
select @@global.sql_mode;
```

去掉里面的 `ONLY_FULL_GROUP_BY` 模式，然后set 回去
```sql
set @@global.sql_mode = "查出来的值，去掉 ONLY_FULL_GROUP_BY"
```

##### 修改配置文件中的 ONLY_FULL_GROUP_BY 模式

找到 `my.ini` 文件，修改其中的 `sql_mode` 值
```sql
sql_mode="查出来的值，去掉 ONLY_FULL_GROUP_BY"
```

##### 借助 and_value() 函数忽略没有参与分组的列
可以使用` ANY_VALUE()` 引用未聚合的列,而无需禁用` ONLY_FULL_GROUP_BY` 即可获得相同的效果。

例如：
```sql
mysql> SELECT name, MAX(age) FROM t;
ERROR 1140 (42000): In aggregated query without GROUP BY, expression
#1 of SELECT list contains nonaggregated column 'mydb.t.name'; this
is incompatible with sql_mode=only_full_group_by
```

借助 `ANY_VALUE()` 改成这样就好了：
```sql
SELECT ANY_VALUE(name), MAX(age) FROM t;
```

参考链接： [MySQL Handling of GROUP BY](https://dev.mysql.com/doc/refman/8.0/en/group-by-handling.html)


最后恰饭 [阿里云全系列产品/短信包特惠购买 中小企业上云最佳选择 阿里云内部优惠券](https://www.aliyun.com/minisite/goods?userCode=0amqgcs9)
---
layout:     post
title:      MySQL 中 delete truncate drop 的区别
subtitle:   MySQL 中 delete truncate drop 的区别
date:       2020-07-17
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - Go
    - PHP
    - MySQL
    - drop
    - delete
    - truncate
---

> 最直观区别：truncate drop 是 DDL 语句，有隐式提交，不可回滚，delete 是 DML 语句，可以回滚。

##### truncate
- truncate 会删除并重新创建表，比 delete 要快，尤其对于大型表。
- truncate 会导致隐式提交，因此无法回滚。
- 如果当前有活跃的表锁，则无法进行 truncate
- 如果表或者 NDB 表的引用表的其他表有任何约束，则他们的 truncate 会失败，而 delete 允许同一表的列之间的外键约束。（没太理解，原文为：TRUNCATE TABLE fails for an table or NDB table if there are any constraints from other tables that reference the table. Foreign key constraints between columns of the same table are permitted. InnoDB FOREIGN KEY
）
- truncate 不返回已删除的函数，通常是 “0行受影响”，应理解为 “无信息”。
- 只要表定义有效，即使数据或索引文件已损坏，也可以将表重新创建为具有 truncate table 的空表。
- 任何值都将重置为其开始值。即使对于并且通常不重用序列值也是如此。
- 与分区表一起使用时,TRUNCATE TABLE保留分区。也就是说，在分区定义不受影响的情况下，将删除并重新创建数据和索引文件。
- truncate 不会调用触发器，delete 会调用触发器。
- truncate 设置支持 INNODB 引擎损坏的表。

##### drop
- 它删除表定义和所有表数据。如果对表进行了分区，则该语句将删除表定义，其所有分区，存储在这些分区中的所有数据以及与删除的表关联的所有分区定义。
- drop table 也会删除该表的所有触发器。
- drop table 是 DDL，也会导致隐式提交，不可回滚，在和 [关键字](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html) 一起提交的情况下例外。
- drop table 不会调用触发器。


##### delete
- delete 是 DML，会记录日志，同时也可以回滚。
- delete 会对行加锁。
- 被删除的数据行只是被标记删除，不会减少占用空间，新增的数据可以直接复用标记为删除的记录的存储空间。


参考链接：
1. [mysql doc truncate table](https://dev.mysql.com/doc/refman/8.0/en/truncate-table.html)
2. [mysql doc drop table](https://dev.mysql.com/doc/refman/8.0/en/drop-table.html)
3. [Difference Between TRUNCATE, DELETE, And DROP In SQL Server](https://www.c-sharpcorner.com/blogs/difference-between-truncate-delete-and-drop-in-sql-server1)


最后恰饭 [阿里云全系列产品/短信包特惠购买 中小企业上云最佳选择 阿里云内部优惠券](https://www.aliyun.com/minisite/goods?userCode=0amqgcs9)
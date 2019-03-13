---
layout:     post
title:      MySQL InnoDB 行存储格式
subtitle:   MySQL InnoDB 行存储格式
date:       2019-03-13
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - MySQL
    - InnoDB
    - 行存储格式
---

`InnoDB`存储引擎支持四名的格式：`REDUNDANT`，`COMPACT`， `DYNAMIC`，和`COMPRESSED`。

**InnoDB行格式概述**

| 行格式       | 紧凑的存储特性 | 增强的可变长度列存储 | 大索引键前缀支持 | 压缩支持 | 支持的表空间类型                 | 所需的文件格式 |
| ------------ | -------------- | -------------------- | ---------------- | -------- | -------------------------------- | -------------- |
| `REDUNDANT`  | 没有           | 没有                 | 没有             | 没有     | system，table-per-table，general | Antelope or Barracuda     |
| `COMPACT`    | 是             | 没有                 | 没有             | 没有     | system，table-per-table，general | Antelope or Barracuda     |
| `DYNAMIC`    | 是             | 是                   | 是               | 没有     | system，table-per-table，general | Barracuda         |
| `COMPRESSED` | 是             | 是                   | 是               | 是       | file-per-table, general                  | Barracuda         |


### REDUNDANT 行格式
`REDUNDANT`格式提供与旧版MySQL的兼容性。

`REDUNDANT`行的格式是由两个支持 `InnoDB`的文件格式（`Antelope`和`Barracuda`）。有关更多信息，请参见[第14.10节“InnoDB文件格式管理”](https://dev.mysql.com/doc/refman/5.7/en/innodb-file-format.html)。

使用`REDUNDANT`行格式的表将可变长度列值（[`VARCHAR`](https://dev.mysql.com/doc/refman/5.7/en/char.html)， [`VARBINARY`](https://dev.mysql.com/doc/refman/5.7/en/binary-varbinary.html)和， [`BLOB`](https://dev.mysql.com/doc/refman/5.7/en/blob.html)和 [`TEXT`](https://dev.mysql.com/doc/refman/5.7/en/blob.html)类型）的前768个字节存储在B树节点内的索引记录中，其余部分存储在溢出页面上。大于或等于768字节的固定长度列被编码为可变长度列，可以在页外存储。例如，`CHAR(255)`如果字符集的最大字节长度大于3，则列可以超过768字节`utf8mb4`。

如果列的值为768字节或更少，则不使用溢出页，并且可能导致I / O的一些节省，因为该值完全存储在B树节点中。这适用于相对较短的`BLOB`列值，但可能导致B树节点填充数据而不是键值，从而降低其效率。具有许多`BLOB`列的表可能导致B树节点变得太满，并且包含的行太少，使得整个索引的效率低于行较短或列值存储在页外的情况。

#### REDUNDANT 行格式存储特性
`REDUNDANT`行格式有如下存储特性：

- 每个索引记录包含一个6字节的标头。标头用于将连续记录链接在一起，以及用于行级锁定。
- 聚簇索引中的记录包含所有用户定义列的字段。此外，还有一个6字节的事务ID字段和一个7字节的滚动指针字段。
- 如果没有为表定义主键，则每个聚簇索引记录还包含一个6字节的行ID字段。
- 每个辅助索引记录包含为聚簇索引键定义的所有主键列，这些列不在辅助索引中。
- 记录包含指向记录的每个字段的指针。如果记录中字段的总长度小于128字节，则指针是一个字节; 否则，两个字节。指针数组称为记录目录。指针指向的区域是记录的数据部分。
- 在内部，固定长度的字符列，例如 [`CHAR(10)`](https://dev.mysql.com/doc/refman/5.7/en/char.html)以固定长度格式存储。尾随空格不会从[`VARCHAR`](https://dev.mysql.com/doc/refman/5.7/en/char.html)列中截断 。
- 大于或等于768字节的固定长度列被编码为可变长度列，可以在页外存储。例如，`CHAR(255)`如果字符集的最大字节长度大于3，则列可以超过768字节 `utf8mb4`。
- SQL `NULL`值在记录目录中保留一个或两个字节。`NULL`如果存储在可变长度列中，则SQL 值将在记录的数据部分中保留零个字节。对于固定长度的列，列的固定长度保留在记录的数据部分中。为`NULL` 值保留固定空间允许将列从`NULL`非`NULL`值更新 到非值，而不会导致索引页碎片。

### COMPACT 行格式

与`REDUNDANT`行格式相比，`COMPACT`行格式减少了大约20％的行存储空间，`REDUNDANT` 代价是增加了某些操作的CPU使用。如果您的工作负载是受缓存命中率和磁盘速度限制的典型工作负载，则`COMPACT`格式可能会更快。如果工作负载受CPU速度限制，则紧凑格式可能会变慢。

`COMPACT`行的格式是由两个支持 `InnoDB`的文件格式（`Antelope`和`Barracuda`）。有关更多信息，请参见[第14.10节“InnoDB文件格式管理”](https://dev.mysql.com/doc/refman/5.7/en/innodb-file-format.html)。

使用`COMPACT`行格式的表将可变长度列值（[`VARCHAR`](https://dev.mysql.com/doc/refman/5.7/en/char.html)， [`VARBINARY`](https://dev.mysql.com/doc/refman/5.7/en/binary-varbinary.html)和， [`BLOB`](https://dev.mysql.com/doc/refman/5.7/en/blob.html)和 [`TEXT`](https://dev.mysql.com/doc/refman/5.7/en/blob.html)类型）的前768个字节存储在[B树](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_b_tree)节点内的索引记录中，其余部分存储在溢出页面上。大于或等于768字节的固定长度列被编码为可变长度列，可以在页外存储。例如，`CHAR(255)`如果字符集的最大字节长度大于3，如果列是 `utf8mb4` 字符类型时可以超过768字节。

如果列的值为768字节或更少，则不使用溢出页，并且可能导致 I/O 的一些节省，因为该值完全存储在B树节点中。这适用于相对较短的`BLOB`列值，但可能导致B树节点填充数据而不是键值，从而降低其效率。具有许多`BLOB`列的表可能导致B树节点变得太满，并且包含的行太少，使得整个索引的效率低于行较短或列值存储在页外的情况。

#### COMPACT 行格式存储特性

`COMPACT`行格式有如下存储特性：

- 每个索引记录包含一个5字节的头，可以在可变长度头之前。标头用于将连续记录链接在一起，以及用于行级锁定。

- 记录头的可变长度部分包含用于指示`NULL`列的位向量。如果索引中的列数可以 `NULL`是*N*，则位向量占用 字节。（例如，如果可以有9到16列的任何位置，则位向量使用两个字节。）不占用此向量中的位以外的空间的列。标题的可变长度部分还包含可变长度列的长度。每个长度需要一个或两个字节，具体取决于列的最大长度。如果索引中的所有列都是`CEILING(*N*/8)``NULL``NULL``NOT NULL` 并且具有固定长度，记录头没有可变长度部分。

- 对于每个非`NULL`可变长度字段，记录头包含一个或两个字节的列长度。如果列的一部分存储在溢出页面的外部，或者最大长度超过255个字节且实际长度超过127个字节，则只需要两个字节。对于外部存储列，2字节长度表示内部存储部分的长度加上指向外部存储部分的20字节指针。内部部分为768字节，因此长度为768 + 20。20字节指针存储列的真实长度。

- 记录头后面是非`NULL`列的数据内容。

- 聚簇索引中的记录包含所有用户定义列的字段。此外，还有一个6字节的事务ID字段和一个7字节的滚动指针字段。

- 如果没有为表定义主键，则每个聚簇索引记录还包含一个6字节的行ID字段。

- 每个辅助索引记录包含为聚簇索引键定义的所有主键列，这些列不在辅助索引中。如果任何主键列是可变长度，则每个辅助索引的记录头都有一个可变长度部分来记录它们的长度，即使在固定长度列上定义了二级索引。

- 在内部，对于非变长字符集，固定长度字符列（例如以 [`CHAR(10)`](https://dev.mysql.com/doc/refman/5.7/en/char.html)固定长度格式存储）。尾随空格不会从[`VARCHAR`](https://dev.mysql.com/doc/refman/5.7/en/char.html)列中截断 。

- 在内部，对于可变长度字符集，例如 `utf8mb3`和`utf8mb4`， `InnoDB`尝试通过修剪尾随空格以字节存储 。如果列值的字节长度 超过字节，则将尾随空格调整为列值字节长度的最小值。 列的最大长度 是最大字符字节长度 × [`CHAR(*N*)`](https://dev.mysql.com/doc/refman/5.7/en/char.html)*N*[`CHAR(*N*)`](https://dev.mysql.com/doc/refman/5.7/en/char.html)*N*[`CHAR(*N*)`](https://dev.mysql.com/doc/refman/5.7/en/char.html)

*N*保留 最少的字节数 。在许多情况下保留最小空间可以在不导致索引页碎片的情况下完成列更新。相比之下，当使用行格式时，列占用最大字符字节长度 × [`CHAR(*N*)`](https://dev.mysql.com/doc/refman/5.7/en/char.html)*N*[`CHAR(*N*)`](https://dev.mysql.com/doc/refman/5.7/en/char.html)*N*`REDUNDANT`

大于或等于768字节的固定长度列被编码为可变长度字段，可以在页外存储。例如，`CHAR(255)`如果字符集的最大字节长度大于3，如果列是 `utf8mb4` 字符类型时可以超过768字节。

##### COMPACT 行格式存储特性图解
MySQL InnoDB COMPAT 行格式结构<br />
![MySQL InnoDB COMPAT 行格式结构](https://alpha2016.github.io/img/2019-03-13-mysql-innodb-compact-format.jpg "MySQL InnoDB COMPAT 行格式")

MySQL InnoDB COMPAT 行格式结构头信息<br />
![MySQL InnoDB COMPAT 行格式头信息](https://alpha2016.github.io/img/2019-03-13-mysql-innodb-compact-header.jpg "MySQL InnoDB COMPAT 行格式头信息")

MySQL InnoDB COMPAT 行格式结构头信息说明<br />
| 名称         | 大小(bit) | 描述                                                         |
| ------------ | --------- | ------------------------------------------------------------ |
| 预留位           | 1         | 未知                                                         |
| 预留位           | 1         | 未知                                                         |
| delete_flag  | 1         | 该行是否已被删除                                             |
| min_rec_flag | 1         | 为1，如果该记录是预先被定义为最小的记录                      |
| n_owned      | 4         | 该记录拥有的记录数                                           |
| heap_no      | 13        | 索引堆中该记录的排序记录                                     |
| record_type  | 3         | 记录类型，000表示普通，001表示B+树节点指针，010表示infimum，011表示supermum，1xx表示保留 |
| next_record  | 16        | 页中下一条记录的相对位置   

实际上，InnoDB 会每条数据加三个隐藏列，分别为

| 列名   | 是否必须 | 占用空间 | 描述                       |
| ------ | -------- | -------- | -------------------------- |
| DB_ROW_ID | 否       | 6字节    | 行ID，**唯一标识一条记录** |
| DB_TRX_ID |	是 |	6字节	| 事务ID |
| DB_ROLL_PTR	| 是 |	7字节	| 回滚指针    |

### DYNAMIC 行格式

`DYNAMIC`行格式提供相同的存储特性的`COMPACT`行格式，但增加了对长可变长度列增强的存储功能，并支持大型索引键的前缀。

Barracuda文件格式支持`DYNAMIC` 行格式。请参见[第14.10节“InnoDB文件格式管理”](https://dev.mysql.com/doc/refman/5.7/en/innodb-file-format.html)。

使用时创建表 `ROW_FORMAT=DYNAMIC`，`InnoDB` 可以完全在页外存储长的可变长度列值（for [`VARCHAR`](https://dev.mysql.com/doc/refman/5.7/en/char.html)， [`VARBINARY`](https://dev.mysql.com/doc/refman/5.7/en/binary-varbinary.html)和， [`BLOB`](https://dev.mysql.com/doc/refman/5.7/en/blob.html)和 [`TEXT`](https://dev.mysql.com/doc/refman/5.7/en/blob.html)类型），聚簇索引记录只包含指向溢出页的20字节指针。大于或等于768字节的固定长度字段被编码为可变长度字段。例如，`CHAR(255)`如果字符集的最大字节长度大于3，如果列是 `utf8mb4` 字符类型时可以超过768字节。

列是否存储在页外是否取决于页面大小和行的总大小。当行太长时，选择最长的列进行页外存储，直到聚簇索引记录适合[B树](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_b_tree)页面。 [`TEXT`](https://dev.mysql.com/doc/refman/5.7/en/blob.html)并且 [`BLOB`](https://dev.mysql.com/doc/refman/5.7/en/blob.html)是小于或等于40个字节的列被存储在线路。

`DYNAMIC`行格式保持存储在它是否适合的索引节点整个行的效率（如做的 `COMPACT`和`REDUNDANT` 格式），但是`DYNAMIC`行格式避免填充B-树节点具有大量长列的数据字节的问题。该`DYNAMIC`行格式是基于这样的思想，如果一个长的数据值的一部分被存储关闭页，它通常是最有效的存储关闭页整个值。对于`DYNAMIC`格式，较短的列可能保留在B树节点中，从而最小化给定行所需的溢出页数。

`DYNAMIC`行格式支持索引键的前缀可达3072个字节。此功能由[`innodb_large_prefix`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_large_prefix)变量控制，该 变量默认启用。有关[`innodb_large_prefix`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_large_prefix)更多信息，请参阅 变量描述。

使用`DYNAMIC`行格式的表可以存储在系统表空间，每表文件表空间和一般表空间中。要`DYNAMIC`在系统表空间中存储表，请禁用 [`innodb_file_per_table`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_file_per_table)和使用常规`CREATE TABLE`或`ALTER TABLE`语句，或将`TABLESPACE [=] innodb_system`表选项与`CREATE TABLE`或一起使用`ALTER TABLE`。在 [`innodb_file_per_table`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_file_per_table)和 [`innodb_file_format`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_file_format)变量不适用于一般的表空间，也没有使用时，它们是适用`TABLESPACE [=] innodb_system` 表选项存储`DYNAMIC`在系统表空间表。

#### DYNAMIC 行格式存储特性

`DYNAMIC`行格式是一个偏差 `COMPACT`行格式。有关存储特性，请参阅 [COMPACT行格式存储特性](https://dev.mysql.com/doc/refman/5.7/en/innodb-row-format.html#innodb-compact-row-format-characteristics)。

### COMPRESSED 行格式

`COMPRESSED`行格式提供相同的存储特性和功能的 `DYNAMIC`行格式，但增加了对表和索引数据压缩的支持。

Barracuda文件格式支持 `COMPRESSED`行格式。请参见 [第14.10节“InnoDB文件格式管理”](https://dev.mysql.com/doc/refman/5.7/en/innodb-file-format.html)。

`COMPRESSED`行格式使用类似的内部细节关闭页存储为`DYNAMIC`行格式，从表和索引数据的附加存储和性能的考虑被压缩，并使用较小的页大小。使用`COMPRESSED`行格式，该`KEY_BLOCK_SIZE`选项控制在聚簇索引中存储的列数据量，以及溢出页面上放置了多少。有关`COMPRESSED`行格式的更多信息 ，请参见 [第14.9节“InnoDB表和页面压缩”](https://dev.mysql.com/doc/refman/5.7/en/innodb-compression.html)。

`COMPRESSED`行格式支持索引键的前缀可达3072个字节。此功能由[`innodb_large_prefix`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_large_prefix)变量控制，该 变量默认启用。有关[`innodb_large_prefix`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_large_prefix)更多信息，请参阅 变量描述。

`COMPRESSED`可以在每个表的文件表空间或通用表空间中创建 使用行格式的表。系统表空间不支持 `COMPRESSED`行格式。要将`COMPRESSED`表存储 在每个表的文件表空间中，[`innodb_file_per_table`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_file_per_table)必须启用该 变量，并且 [`innodb_file_format`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_file_format)必须将其设置为 `Barracuda`。在 [`innodb_file_per_table`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_file_per_table)和 [`innodb_file_format`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_file_format)变量不适用于一般的表空间。一般表空间支持所有行格式，但需要注意的是，由于物理页面大小不同，压缩和未压缩表不能在同一个通用表空间中共存。有关更多信息，请参见 [第14.6.3.3节“常规表空间”](https://dev.mysql.com/doc/refman/5.7/en/general-tablespaces.html)。

#### COMPRESSED 行格式存储特性

`COMPRESSED`行格式是一个偏差 `COMPACT`行格式。只不过在处理行溢出数据时有点儿分歧，不会在记录的真实数据处存储字符串的前768个字节，而是把所有的字节都存储到其他页面中，只在记录的真实数据处存储其他页面的地址。另外，`Compressed` 行格式会采用压缩算法对页面进行压缩。

参考链接：
1. [MySQL InnoDB row format docs](https://dev.mysql.com/doc/refman/5.7/en/innodb-row-format.html "MySQL InnoDB 行格式文档")
2. [掘金小册](https://juejin.im/book/5bffcbc9f265da614b11b731 "小孩子的小册")
3. [InnoDB 行记录格式](http://zhongmingmao.me/2017/05/07/innodb-table-row-format/ "zhongmingmao博客文章")

> 文章当作笔记，版权为小册作者及 MySQL 官方文档

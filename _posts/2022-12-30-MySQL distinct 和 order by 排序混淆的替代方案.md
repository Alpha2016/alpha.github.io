场景是：从一堆学习记录中，去重并获取最近学习的几条课程ID，随手就能想到这样的一条SQL语句：
```sql
select distinct a from table order by updated_at desc limit 5
```
如果列为 a 的数据有很多条，就会发现最终取到的那条数据可能不是 updated_at 最近的那条数据，因为 distinct 有一次默认的排序，然后生成一个临时表，
然后 order by 无法从最开始的原始数据中进行排序，仅排序中间表，无法得出正确结果。改成 distinct a, updated_at 的话，
实际上又失去了 distinct 的意义了。

方案一：
使用子查询方式，将结果先排序，当做一个表，然后去重保留最新的一条数据
```sql
select distinct a from (select a from table order by updated_at desc) t limit 5
```

方案二：
借助 max 和 group by 特性直接取最大值，取值
```sql
select a, max(updated_at) from table group by a order by updated_at desc limit 5
```

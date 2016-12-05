---
title: 高性能 MySQL 笔记-高级
date: 2016-12-05 21:30:21
tags:
---

### 分区表
分区表适用于表非常大以至于无法全部放入内存中，或者表中只有部分热点数据的场景。一个表最多只能有 1024 个分区，分区表达式必须返回整数，分区表无法使用外键约束。

#### 语法

    create table xxx(
        -- some columns
    ) engine=innodb partition by range( xxxx )(
        partition table_1 values less than ( a_value ),
        partition table_2 values less than ( a_value ),
        partition table_3 values less than ( a_value )
    )

几种分区模式：
* Range（范围） 
* Hash（哈希） 
* Key（键值） 
* List（预定义列表）
* Composite（复合模式）

什么情况下会出问题
* null 值使得分区过滤无效，null 值会被放在第一个分区。
* 分区列和索引列不匹配
* 选择分区成本过高
* 打开并锁住所有底层表可能成本很高(在选择分区时)

### 总结的几个原则
1. 不用 Merge 表，这是一种即将淘汰的技术，有很多缺陷。
2. 如果打算使用视图提升性能最好做详细的测试，视图的性能很难预测。mysql 不支持物化视图，也不支持视图中创建索引。
3. 外键约束会带来很大的性能消耗，如果只是使用外键做约束，通常在应用程序中实现会更好。
4. 存储过程、触发器、存储函数、事件这些特性在主从复制，基于语句的复制场景下还容易出现问题，所以这些特性虽然能节省网络开销，但还是应该少用。
5. 很少会用到 xa 事务特性，最好不用去修改 xa 相关配置。
6. 查询缓存在高并发查询环境中会导致性能下降，最好不要使用，而应该使用 memcached 等其他解决方案。

### mysql 字符集校对规则
_cs  代表大小写敏感方式比较字符串
_ci  大小写不敏感
_bin 二进制

### 


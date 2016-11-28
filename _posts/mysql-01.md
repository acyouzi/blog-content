---
title: 高性能 MySQL 笔记 01
date: 2016-11-27 23:12:24
tags:
---

### 基础

1. 锁的粒度，表级锁是 mysql 中最基本的锁策略，是一个读写分离的锁。行级锁是在 mysql 的存储引擎层实现，没有在服务器层实现，innodb 中实现了行级锁。
2. 隔离级别
    * READ UNCOMMITTED 未提交读，会发生脏的
    * READ COMMITTED 提交读，大多数数据库的默认隔离级别，会发生不可重复读
    * REPEATABLE READ 可重复读，会发生幻读
    * SERIALIZABLE 串行化

    MySQL 能识别全部四个隔离级别，Innodb 引擎也能支持所有的隔离级别。可用通过 set transaction isolation level xxxx 来设置隔离级别。
3. 多版本并发控制(MVCC)，在很多情况下可以避免加锁操作，Innodb 实现了MVCC, 他通过在每行记录后面保存两个隐藏的列来实现，这两个列一个保存了行的创建版本号，一个保存了列的过期版本号。MVCC 只在 REPEATABLE READ 和 READ COMMITTED 两个级别下工作。

### InnoDB
1. 采用 MVCC 来支持高并发
2. 实现了默认的四个标准隔离级别，REPEATABLE READ 是默认隔离级别
3. 通过间隙锁策略防止幻读
4. InnoDB 表基于聚簇索引建立，主键查询性能很高，数据的存储格式是平台独立的。

### 转换表存储引擎的方法
1. alter table table_name engine = innodb ,会按行将数据从原表复制到一张新表，如果数据表很大会消耗很长时间。
2. 使用 mysqldump 将数据导出到文件，然后修改建表语句
3. 使用 sql 语句复制表

    create table innodb_table like myisam_table;
    alter table innodb_table engine = innodb;
    insert into innodb_table select * from myisam_table;

### 数据类型选择
1. 如何选择数据类型
    * 选择更小的
    * 选择简单数据类型，例如用 data,time,datatime 来存储日期和时间而不是使用字符串。用整形存储IP( 使用 inet_aton() 和 inet_ntoa() 完成ip和整形之间的转换)
    * 尽量避免 Null，可为 Null 的列使得索引、索引统计和值比较都变得复杂，会占用更多的存储空间。如果计划在列上建立索引应该避免设计成可为 Null 的列。
2. 整数类型，包括 TINYINT（8b） SMALLINT(16b) MEDIUMINT(24b) INT(32b) BIGINT(64b)，可以通过 USIGNED 属性标示无符号属性。
    mysql 可以为整数类型指定宽度，如int(9),但是这没什么意义。它不会限制值的宽度，只是规定了mysql的一些交互工具显示字符的个数。
3. 实数类型，FLOAT DOUBLE，还有 DECIMAL。DECIMAL 用于存储精确小数，最多只能存储 65 个数字。 mysql 可以指定浮点数所需要的精度，但是最好只指定数据类型而不指定精度。
    可以考虑使用 BIGINT 替代 DECIMAL,这样可以避免 DECIMAL 精确计算所付出的代价。
4. 字符串类型
    * varchar 用于存储变成字符串，需要1-2个额外字节记录字符串的长度，如果列的最大长度小于等于255字节，使用一个字节标示，否则使用两个字节。
    * char ，当存储 char 值时, mysql会删除所有的末尾空格。 例如 '  str  ' 存到 char 类型的列中变为 '  str', varchar 不会删除末尾空格。
    * Memory 引擎只支持定长的行，所以即使变长的字符串也会分配定长的空间
    * binary varbinary 存储二进制字符串，binary 在填充时采用 '\0' 并且在检索时也不会去掉填充值。
    * 更长的列会消耗更多的内存，所以虽然 varchar(5) 和 varchar(200) 在存储一个 hello 时空间开销是一样的，但是 varchar(5) 会消耗更少的内存，因为mysql通常会分配固定大小的内存保存内部值，尤其是使用内存临时表时进行排序时会特别糟糕。
5. blob 和 text 用来存储很大的数据，blob 存储二进制，text 存储字符。更具体分为 tinytext, smalltext, text, mediumtext, longtext。
    mysql 把 blob 和 text 当做一个单独的对象处理，当值太大时，innodb 会使用单独的空间存储，此时每个值内存储一个指针。

    Memory 引擎不支持 blob text 类型，如果查询使用了blob或者text列，并且使用了隐式临时表，将不得不使用 myisam 磁盘临时表。可以在用到 blob 的地方使用 substring(column,length) 函数将列转换为字符串，但是要保证字符串够短，使得临时表大小不超出 max_heap_table_size 或 tmp_table_size, 否则会转化为磁盘临时表。
6. 时间日期
    * datetime 按照 YYYYMMDDHHMMSS 的格式存储到整数中。
    * timestamp 只使用四个字节的存储空间，最多只能表示到 2038 年。可以使用 from_unixtime unix_timestamp 在 timestamp 和日期之间转化。

    应该尽可能使用timestamp，因为它比datetime更高效。同时不推荐把 timestamp 存储为整数值。
7. 标识列选择 
    * 整数通常是最好的标识符，因为他们很快，并且可以自增。
    * 应该避免字符串类型作为标识列，因为他们很耗空间
    * 不要使用完全随机的数字或字符串，会导致 insert 以及一些 select 语句变慢。

### schema 设计陷阱
1. 太多的行，服务器层和存储引擎层之间通过行缓冲格式拷贝数据，然后在服务器层解码成各个列，如果列太多，转换的代价会非常高。
2. 太多关联，如果希望查询执行快速并且并发性好，单个查询最好在12个表以内做关联。
3. null 改用还得用，有时候可能用 -1 之类的值标示空缺业务逻辑会比 null 复杂很多。

### 范式和反范式
1. 范式更新操作通常比反范式快，表通常更小，可以更好地存在内存里面
2. 反范式，不需要做关联，最差的情况也就是全表扫描。

### 计数器表设计
1. 为了使计数器更新操作并行执行，可以在计数器中预先插入多列，更新时随机选取一列，这样可以把一列上的负载分摊开来

        create table my_counter()
            slot TINYINT unsigned not null primary key,
            counter int unsigned not null
        )ENGINE=innodb

    预先插入 100 行数据，counter = 0,更新语句如下

        update my_counter set counter = counter + 1 where slot = RAND() * 100;
    
    如果需要每隔一段时间开始一个新的计数器，可以如下设计：

        create table daily_counter( 
            day date not null,
            slot TINYINT unsigned not null,
            counter int unsigned not null,
            primary key(day,slot)
        )ENGINE=innodb

        // 更新语句：
        insert into daily_counter(day,slot,counter) values(CURRENT_DATE,RAND()*100,1) on duplicate key update counter = counter + 1;

### 只修改 .frm 文件
移除一个列的 auto_increment 属性，增加，移除，或者更改ENUM和SET常量可能不需要重建表：

    create table xxx_new like xxx;
    alter table xxx_new modify column xxx;
    flush table with read lock;

    cp xxx.frm xxx.frm.bak
    cp xxx_new.frm xxx.frm

    unlock tables;
    show columns from xxx \g;




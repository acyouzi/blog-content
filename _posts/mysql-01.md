---
title: 高性能 MySQL 笔记-基础
date: 2016-11-27 23:12:24
author: "acyouzi"
# cdn: header-off
description: 
tags:
    - mysql
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
3. null 该用还得用，有时候可能用 -1 之类的值标示空缺业务逻辑会比 null 复杂很多。

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

### 索引
mysql 中，索引是在存储引擎层实现而不是服务器层，所以没有统一的索引标准。

#### B-Tree 索引
实际上很多存储引擎（如 InnoDB）使用的是 B+Tree, 只不过MySQL 中使用 B-Tree 作为关键字，B-Tree 对数据列是顺序组织存储的，所以很适合范围查找。

建立多值索引时，索引对多个值排序的依据是 create table 语句中定义索引时列的顺序，B-Tree 索引适用于全键值、键值范围、最左前缀查找。因为索引树中节点是有序的，索引还可以用于查询中的 order by 操作。

#### 哈希索引
mysql 中只有 Memory 引擎显式支持哈希索引，这也是 Memory 的默认索引类型，而且 Memory 引擎支持的是非唯一的哈希索引。 Memory 引擎同时支持非唯一索引。

InnoDB 中有一种特殊的功能叫做自适应哈希索引，当InnoDB 注意到某些索引值被使用的非常频繁时，他会在内存中基于 B-Tree 之上再创建一个哈希索引。

#### 自建哈希索引
在数据库中建立新建一列保存哈希函数的哈希值，可以使用 CRC32 函数作为 hash 函数，然后在select 语句中拼接上 and xxx_crc=CRC32("info") 

### 高性能索引策略
1. 独立的列，在 where 子句中索引不能是表达式的一部分，也不能是函数的参数。
2. 前缀索引，索引很大的字符串时会降低性能，通常可以索引开始的部分字符串,建立前缀索引需要选择一个可选择性比较合适的列的长度，使用如下公式计算：

        select count( distinct left(city,5) )/count(*) from xxx;
        // 建立索引
        alter table xxx add key (my_col(5))

3. 多列索引，建立多个单独的索引不是好策略，最好建立一个包含多个列的索引，通过 explain 语句查看 sql 执行如果在 extra 列中见到 Using union 等合并策略可能说明索引列建的很糟糕。
4. 选择合适的索引列顺序，要考虑可选择性，同时也要查询条件的具体值（值的分布）有关。不要假设平均情况下的性能等于最坏的情况，最坏的情况可能很坏，可以考虑也谢数据库优化以外的方法避免最坏情况。
5. 聚簇索引，是一种数据存储方式，所以一个表只能有一个聚簇索引。
    
    InnoDB 使用主键来做聚簇索引，如果没有主键会选择一个唯一的非空索引代替，如果没有这样的索引会隐式定义一个主键来做聚簇索引。InnoDB 只聚集在同一个页面中的数据，所以包含相邻主键的数据可能相聚甚远。

6. 覆盖索引,如果一个索引包含所有需要查询的字段值，我们就称之为覆盖索引。当发起一个呗索引覆盖的查询时，在 explain 的 extra 列可以见到 "Using index" 的信息。
7. 使用索引扫描来做排序. MySQL 有两种方式生成有序结果：通过排序操作和按索引顺序扫描。如果 explain 出来的 type 列值是 "index" 说明使用了索引扫描来做排序。
    只有当索引的列顺序和 order by 子句顺序完全一致，并且所有列的排序方向完成一致，mysql才能使用索引结果做排序，如果需要查询需要关联多张表，则只有当 order by 子句引用字段全为第一个表时才能使用索引排序。
8. 未使用索引，运行一段时间，然后查询 infomation schema.index statistics 查询每个索引的使用频率，然后可以删除不常用的索引。

### 聚簇索引 
是一种数据物理组织方式，InnoDB 使用主键来做聚簇索引.

#### 聚簇索引的优点
1. 可以把相关数据保存在一起。
2. 数据访问更快，将索引和数据块保存在同一个 B-Tree 中。
3. 使用索引覆盖查询可以直接使用叶节点中的主键值。

#### 聚簇索引的缺点
1. 插入速度依赖插入顺序，如果不按照主键顺序插入数据那就呵呵了。
2. 更新聚簇索引代价很高。
3. 页分裂问题，导致数据存储不连续
4. 二级索引的叶子节点保存了引用行的主键，可能导致二级索引变大。
5. 二级索引访问需要两次索引查找（因为二级索引叶子节点保存的是行的主键，而不是物理位置，所以需要再通过主键查找）

#### InnoDB 主键选择
如果正在使用 InnoDB 并且没有什么数据需要聚集，使用一个自增长的代理主键，这样可以保证数据行是按顺序写入，对于主键做关联操作性能也会更好。

在高并发场景下，InnoDB中按主键顺序插入可能会造成明显的争用。一个是主键上界成为插入热点，另一个热点可能是自增长的锁机制。

### in() 语句与范围条件
查询只能使用索引的最左前缀，直到遇到第一个范围条件列。假设有索引 ( sex , country , age ) 希望查询 country = china and age between 18 and 25，为了能用上前面的索引，可以加上条件 sex in('M','F').

### JOIN
JOIN 按照功能大致分为如下三类：

1. INNER JOIN（内连接）：取得两个表中存在连接匹配关系的记录。

        // 下面两句结果相同
        SELECT article.aid,article.title,user.username FROM article INNER JOIN user ON article.uid = user.uid
        SELECT article.aid,article.title,user.username FROM article,user WHERE article.uid = user.uid

        SELECT article.aid,article.title,user.username FROM article CROSS JOIN user

    CROSS JOIN 与 INNER JOIN 的表现是一样的，在不指定 ON 条件得到的结果都是笛卡尔积，反之取得两个表完全匹配的结果。同时 INNER JOIN 与 CROSS JOIN 可以省略 INNER 或 CROSS 关键字。

2. LEFT JOIN（左连接）：取得左表（table1）完全记录，即使右表（table2）并无对应匹配记录。

        SELECT article.aid,article.title,user.username FROM article LEFT JOIN user ON article.uid = user.uid

3. RIGHT JOIN（右连接）：与 LEFT JOIN 相反，取得右表（table2）完全记录，即是左表（table1）并无匹配对应记录。

        SELECT article.aid,article.title,user.username FROM article RIGHT JOIN user ON article.uid = user.uid

4. JOIN 还支持多表连接
5. MySQL 没有提供 SQL 标准中的 FULL JOIN（全连接）。
6. Mysql 中联接SQL语句中，ON子句的语法格式为：table1.column_name = table2.column_name，当模式设计对联接表的列采用了相同的命名样式时，就可以使用 USING 语法来简化 ON 语法，格式为：USING(column_name)。

### 分析低效查询
对于低效查询通过以下两个步骤分析很有效：
* 确认程序是否在检索大量超过需要的数据
    1. 查询了不需要的数据，使用 limit 限制.
    2. 多表关联时返回全部列( select * from ... inner join ..)
    3. 总是取出全部列 ( select * from ... 这种写法能提高代码的可复用性，但是会影响性能，这是需要权衡的地方)

* 确认 MySQL 服务器层是否在分析大量超过需要的数据行
    主要是 where 子句可能不能使用索引，而导致从数据表中返回数据，然后过滤不满足条件的记录( explain 的 extra 列出现 Using Where).

Where 子句有如下三种使用方式：

1. 索引中使用 where 条件来过滤不匹配的记录
2. 使用索引覆盖扫描( explain 的 extra 列出现 Using index)
3. 从数据表中返回数据，然后过滤不满足的条件

### 重构查询方式
1. 大查询分解为小查询
2. 分解关联查询，很多高性能应用都会对关联查询进行分解，对每个表进行一次单表查询，然后将结果在引用程序中做关联。
    分解关联查询有如下优势：
    * 让缓存的效率更高，缓存单表查询对应的结果对象
    * 将查询分解后，执行单个查询可以减少锁竞争
    * 可以减少冗余记录查询

### 查询执行
1. 客户端发送一条查询给服务器
    mysql 服务器和客户端之间的通信协议是半双工的，在任何时刻，要么客户端向服务器发送数据，要么服务器向客户端发送数据，这两个动作不能同时发生。
2. 查询缓存
    在解析查询语句前，如果查询缓存是打开的，那么 mysql 会优先检测这个产线是否命中查询缓存中的数据。这个查找是大小写敏感的 hash 查找。如果命中缓存在返回查询结果之前 mysql 会再检查一次用户权限。
3. 查询优化处理
    Mysql 使用基于成本的优化器，能够处理的优化类型有：重新定义关联顺序，将外连接转换为内连接，优化 count()、min()、max()（索引列上优化）、索引覆盖查询、子查询优化、提前终止查询、等值传播 ... 
4. 查询执行引擎
5. 返回结果给客户端

### 特定类型优化
1. 优化 count() ， count 在统计列值时要求列值非空(不统计 NULL)，希望统计结果集的行数使用 count(*) 而不是固定到某一列上，这样写意义清晰，性能也会更好。
2. 优化关联查询，保证 using 或者 on 子句上有索引，确保 group by 或 order by 表达式上只涉及表中的一个列。
3. 优化子查询，尽量使用关联查询代替。
4. 优化 limit 语句，如果偏移量太大，效率会非常低，可以考虑尽可能使用索引覆盖查询，通过延迟关联，先在使用覆盖索引查询选出部分列，然后再关联。
        
        select film_id, discription, from film order by title limit 50,5;
        // 可以改写为
        select film.film_id, film.discription
        from film 
            inner join 
                (
                    select film_id from film order by title limit 50,5
                ) as lim 
            using (film_id);

5. 优化 union ，很多优化策略在子句中都没法很好的使用，经常需要手动将一些子句下推到 union 的各个子查询中。union 子句中默认是去重复的，如果不需要去重复需要使用 union all.



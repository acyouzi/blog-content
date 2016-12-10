---
title: 高性能 MySQL 笔记-高级
date: 2016-12-05 21:30:21
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- MySQL
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

### 应用层优化
1. 添加页面缓存模块，缓存部分页面
2. 打开 gzip 压缩，对于现在的 cpu 而言这样的代价很小，但是可以节省大部分流量。
3. 不要为长距离连接配置 keep-alive 通过代理服务器与客户端保持长连接，ap 服务器与代理服务器之间不保持长连接。

#### 缓存控制策略
1. TTL 存活时间，设置一个存活时间，超过这个时间清理数据，适用于数据很少变更或者没有新数据的情况。
2. 显示失效，如果不能接受脏数据，那么在更新原数据时同时使得缓存数据失效，一般有：写-失效 和 写-更新 两种方式实现。
3. 读时失效，在更改旧数据时，为避免脏数据可以在缓存中保存一些信息，当在缓存中读数据时可以利用这些信息判断数据是否失效。(对象版本控制)

### mysql 配置优化
1. socket 文件和 pid 文件不要放在默认的位置，最好明确指定存放位置，否则会发生错误
        
        socket = "/var/lib/mysql/mysql.sock"
        basedir = "/var/lib/mysql" 
        pid_file = "mysql.pid"

2. 使用 innodb 引擎，需要配置 p
        
        innodb_data_home_dir = "/var/lib/mysql/data"
        ## 10M 是文件的初始大小，autoextend 是自动扩展
        innodb_data_file_path = ibdata1:10M:autoextend 
        ## 每个表单独存储一个文件
        innodb_file_per_table = 1
        
        ## 内存的 50 - 80 %
        innodb_buffer_pool_size = 2048M
        ## innodb_buffer_pool_size 的 25%
        innodb_log_file_size = 512M
        ## 通常不需要设置的非常大， 1M - 8M 之间就可以
        innodb_log_buffer_size = 8M
        
        ## 0 把日志缓冲写到日志文件，并且每秒钟刷新一次，但是事务提交时不做任何操作。
        ## 1 默认的，并且是最安全的设置，每次事务提交都刷新到持久化存储，保证不会丢失任何已经提交的事务。
        ## 2 每次提交把日志缓冲写到日志文件，但是并不刷新，是 0 与 1 的折中。
        innodb_flush_log_at_trx_commit = 1

3. log

        log_error = /var/lib/mysql/mysql-error.log
        ## 是否开启慢查询日志，1表示开启，0表示关闭
        slow_query_log = 1
        slow_query_log_file=/var/lib/mysql/mysql-slow.log
        ## 慢查询阈值，当查询时间多于设定的阈值时，记录日志
        ## long_query_time
        ## 未使用索引的查询也被记录到慢查询日志中
        ## log_queries_not_using_indexes
        ## 日志存储方式。log_output='FILE'表示将日志存入文件，默认值是'FILE'。
        ## log_output='TABLE'表示将日志存入数据库
        ## log_output：
        
4. 其他配置
        
        default_storage_engine = InnoDB 
        ## 现代操作系统句柄开销都很小，这个应该设置的尽可能的大
        open_files_limit = 65535
        ## tmp_table_size max_heap_table_size 是用于设置使用 Memory 引擎的内存临时表能使用多大的内存，超过这个就转而使用磁盘临时表
        tmp_table_size = 32M
        max_heap_table_size = 32M
        ## 禁用查询缓存
        query_cache_type = 0
        query_cache_size = 0
        ## 默认是 100 对大多数应用程序来说应该都是不够用的，根据实际业务调整
        max_connections = <..>

        thread_cache = <...>
        open_files_limit = <...>

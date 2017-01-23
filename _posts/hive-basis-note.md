---
title: hive 基础笔记
date: 2017-01-23 13:55:40
tags:
    - hive
    - 大数据
---
## hive 架构
hive 是数据仓库，在 hive 中把表映射为 hdfs 的文件，把查询映射为 hadoop mapreduce

hive 中的元数据可以存储在数据库中，可以存在 mysql 中，Hive 中的元数据包括表的名字，表的列和分区及其属性，表的属性（是否为外部表等），表的数据所在目录

hive 和 hadoop 的关系
![image](hive-hadoop.png)

## 几个概念
> db：在hdfs中表现为${hive.metastore.warehouse.dir}目录下一个文件夹

> table：在hdfs中表现所属db目录下一个文件夹,删除表后, hdfs上的文件都删了

> external table：外部表, 与table类似，不过其数据存放位置可以在任意指定路径, External外部表删除后, hdfs上的文件没有删除, 只是把文件删除了。

> 从外部导入的一些表，不希望被修改，通过外部表对接，内部运行中生成的各种表用内部。

> partition：分区在hdfs中表现为table目录下的子目录

> bucket：桶, 在hdfs中表现为同一个表目录下根据hash散列之后的多个文件, 会根据不同的文件把数据放到不同的文件中 

## hive 安装
1. 修改 conf/hive-env.sh 的 $HADOOP_HOME
2. vim conf/hive-site.xml

        <configuration>
            <property>
            <name>javax.jdo.option.ConnectionURL</name>
            <value>jdbc:mysql://acyouzi01:3306/hive?createDatabaseIfNotExist=true</value>
            <description>JDBC connect string for a JDBC metastore</description>
            </property>
        
            <property>
            <name>javax.jdo.option.ConnectionDriverName</name>
            <value>com.mysql.cj.jdbc.Driver</value>
            <description>Driver class name for a JDBC metastore</description>
            </property>
            
            <property>
            <name>javax.jdo.option.ConnectionUserName</name>
            <value>root</value>
            <description>username to use against metastore database</description>
            </property>
            
            <property>
            <name>javax.jdo.option.ConnectionPassword</name>
            <value>root</value>
            <description>password to use against metastore database</description>
            </property>
        </configuration>

3. schemaTool -initSchema -dbType mysql
4. 运行 hive 命令，进入命令行
    
        可以 hive -e 'filename' 直接运行的写好的 sql 语句

5. Hive thrift服务方式启动，hive 发布为一个服务通过客户端连接

        // 启动
        nohup bin/hiveserver2 1>/var/log/hiveserver.log 2>/var/log/hiveserver.err &
        
        // 连接
        beeline
        beeline> !connect jdbc:hive2://mini1:10000
        然后要求输入用户名，就是你启动服务的那个用户的用户名
    
    如果使用 root 用户登录，发生权限错误，在 core-site.xml 文件中加入,注意是 core-site.xml
        
        <property>
          <name>hadoop.proxyuser.root.hosts</name>
          <value>*</value>
         </property>
         <property>
          <name>hadoop.proxyuser.root.groups</name>
          <value>*</value>
        </property>
        
    并重启 yarn --- 注意，重启 配置文件同步到集群中的所有文件，注意是重启 yarn
    
    另外还有一个配置，如果发生问题可以试一下 
        
        // hdfs-site.xml
        <property>
          <name>dfs.permissions</name>
          <value>false</value>
        </property>

    
### HiveServer2  配置
    
    <configuration>
        <property>
            <name>beeline.hs2.connection.hosts</name>
            <value>localhost:10000</value>
        </property>
        <property>
            <name>beeline.hs2.connection.principal</name>
            <value>hive/dummy-hostname@domain.com</value>
        </property>
        <property>
            <name>beeline.hs2.connection.user</name>
            <value>hive</value>
        </property>
        <property>
            <name>beeline.hs2.connection.password</name>
            <value>hive</value>
        </property>
    </configuration>

### 使用 JDBC 连接

    Class.forName("org.apache.hive.jdbc.HiveDriver");
    Connection cnct = DriverManager.getConnection("jdbc:hive2://<host>:<port>", "<user>", "<password>");
    Statement stmt = cnct.createStatement();
    ResultSet rset = stmt.executeQuery("SELECT foo FROM bar");

## Hive SQL

### 常用数据类型
    
    // 数学
    TINYINT (1-byte from -128 to 127)
    SMALLINT (2-byte from -32,768 to 32,767)
    INT/INTEGER (4-byte from -2,147,483,648 to 2,147,483,647)
    BIGINT (8-byte from -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807)
    FLOAT
    DOUBLE 
    DECIMAL
    
    // Date/Time
    TIMESTAMP
    DATE
    
    // String Types
    STRING 一般只用这一个
    VARCHAR
    CHAR
    
    // Complex Types
    arrays: ARRAY<data_type> 
    写到文件中默认格式是 ['aa','bb']
    maps: MAP<primitive_type, data_type>
    写到文件中默认格式是 {'aa':'bb','cc':'dd'}
    
    
### 可以进行的数学运算
    
    算数运算,逻辑运算,关系运算,按位操作
        
    // 几个例子
    A [NOT] BETWEEN B AND C 
    A IS NULL
    A IS NOT NULL
    A [NOT] LIKE B
    A RLIKE B 如果 A 中的一个子串能匹配正则表达式，返回 True
    
    A AND B 
    A [NOT] IN (val1, val2, ...)
    [NOT] EXISTS (subquery)
    
    A||B 字符串连接
    
### 复合类型创建，操作,集合函数
    
    // 创建
    map(key1, value1, key2, value2, ...)
    array(val1, val2, ...)
    // 操作
    A[n]   A 是 array
    M[key] M 是 map
    // 函数
    size(Map<K.V>)
    size(Array<T>)
    map_keys(Map<K.V>) : returnType array<K>
    map_values(Map<K.V>) : returnType array<V>
    array_contains(Array<T>, value) : returnType boolean
    sort_array(Array<T>) : returnType array<T>

### 数学函数若干
正数负数绝对值，正弦余弦随机数...

### 日期函数若干
时间戳：unix_timestamp from_unixtime 
根据日期获取：year quarter month day hour weekofyear
两个日期相差几天 ：datediff
... 

### 字符串操作函数
拼接转换，json 解析，长度计算， 替换，编码，解码

json 解析

    get_json_object(src_json.json, '$.owner')
    
### 聚合操作

    count(*), count(expr), count(DISTINCT expr[, expr...])
    sum(col), sum(DISTINCT col)
    avg(col), avg(DISTINCT col)
    min(col)
    max(col)
    variance(col), var_pop(col) 方差
    var_samp(col) 无偏采样方差
    stddev_pop(col) stddev_samp(col) 标准差
    covar_pop(col1, col2) covar_samp(col1, col2) 协方差
    corr(col1, col2) 皮尔逊相关系数

### Create Table
    
    CREATE [TEMPORARY] [EXTERNAL] TABLE [IF NOT EXISTS] [db_name.]table_name
        [(col_name data_type [COMMENT col_comment], ...)]  // 列
        [PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)]  // 分表
        [
        [ROW FORMAT row_format] 
        [STORED AS file_format]
         | STORED BY 'storage.handler.class.name' [WITH SERDEPROPERTIES (...)]  -- (Note: Available in Hive 0.6.0 and later)
        ]
        [LOCATION hdfs_path]
    
    CREATE [TEMPORARY] [EXTERNAL] TABLE [IF NOT EXISTS] [db_name.]table_name
        LIKE existing_table_or_view_name
        [LOCATION hdfs_path];
    
    列的数据类型有
        基本类型
        ARRAY < data_type >
        MAP < primitive_type, data_type >
        STRUCT < col_name : data_type [COMMENT col_comment], ...>
        UNIONTYPE < data_type, data_type, ... >
        
    row_format
        DELIMITED   [FIELDS TERMINATED BY char [ESCAPED BY char]]  // 列分隔符
                    [COLLECTION ITEMS TERMINATED BY char] // 集合分隔符
                    [MAP KEYS TERMINATED BY char] 
                    [LINES TERMINATED BY char] // 行分隔符
                    [NULL DEFINED AS char]
        | SERDE serde_name [WITH SERDEPROPERTIES (property_name=property_value, property_name=property_value, ...)]
    
    file_format:
        默认是 TEXTFILE
        还可以存储为 AVRO，ORC，PARQUET 等
        
LOCATION 创建外部表时指定外部表位置
#### BucketedTables 桶

    CREATE TABLE user_info_bucketed(user_id BIGINT, firstname STRING, lastname STRING)
    COMMENT 'A bucketed copy of user_info'
    PARTITIONED BY(ds STRING)
    CLUSTERED BY(user_id) INTO 256 BUCKETS;

指定桶的个数，把一个表分成多个，便于数据的并行处理

#### Row Formats & SerDe

    // json
    ROW FORMAT SERDE 
    'org.apache.hive.hcatalog.data.JsonSerDe' 
    STORED AS TEXTFILE
    // csv
    ROW FORMAT SERDE 
    'org.apache.hadoop.hive.serde2.OpenCSVSerde'
    STORED AS TEXTFILE

#### 分区表
可以指定多个用于分区的列，这些列不在我们定义的表中

    create table table_name (
      id                int,
      dtDontQuery       string,
      name              string
    )
    partitioned by (date string) // data 不在上面定义的列中

### 删除表

    DROP TABLE [IF EXISTS] table_name [PURGE]
    
    不加 PURGE 删除的表进入 .Trash/Current
    否则直接删除

### Show

    Show Databases
    Show Tables
    SHOW PARTITIONS table_name
    SHOW CREATE TABLE ([db_name.]table_name|view_name);
    SHOW COLUMNS (FROM|IN) table_name [(FROM|IN) db_name];
    DESC formatted table_name

### Load

    LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] INTO TABLE tablename [PARTITION (partcol1=val1, partcol2=val2 ...)]
    
    往分桶表 load 数据不会进行分桶操作，只是把数据放到表文件夹下，所以分桶表没有起到作用，应该通过hive数据库中别的表去 insert select 往 分桶表中插入数据

LOCAL 用于说明文件是在本地还是在 hdfs上

OVERWRITE 说明是覆盖还是追加写入

### Insert

    INSERT INTO TABLE tablename [PARTITION (partcol1[=val1], partcol2[=val2] ...)] VALUES ( value [, value ...] ) [, ( value [, value ...] ) ...]
    
    这种方式插入，每个条语句保存在一个文件中(hdfs 只保存数据不修改数据)，一般都是直接拷贝
    
#### 查询结果保存到本地或 hdfs

    INSERT OVERWRITE [LOCAL] DIRECTORY directory1 select_statement1 
    
### export 导出表

    EXPORT TABLE tablename [PARTITION (part_column="value"[, ...])]
    TO 'export_target_path' [ FOR replication('eventid') ]
    
### import 导入表

    IMPORT [[EXTERNAL] TABLE new_or_original_tablename [PARTITION (part_column="value"[, ...])]]
    FROM 'source_path'
    [LOCATION 'import_target_path']

### Select 

#### orderby 子句

    colOrder: ( ASC | DESC )
    colNullOrder: (NULLS FIRST | NULLS LAST)
    orderBy: ORDER BY colName colOrder? colNullOrder? (',' colName colOrder? colNullOrder?)*

#### sortby 子句

    colOrder: ( ASC | DESC )
    sortBy: SORT BY colName colOrder? (',' colName colOrder?)*

#### orderby sortby 的区别
orderby 保证全局有序.
sort 仅仅保证每一个 reduce 有序，如果有多个 reduce 不能保证全局有序

#### Join

    SELECT a.* FROM a JOIN b ON (a.id = b.id)
    SELECT a.* FROM a LEFT OUTER JOIN b ON (a.id <> b.id)
    SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key2)

#### 子查询

    SELECT ... FROM (subquery) name ...
    SELECT ... FROM (subquery) AS name ... 
    // 例子
    SELECT col
    FROM (
      SELECT a+b AS col
      FROM t1
    ) t2
    
hive 子查询还有一些语法不支持

### Hive 自定义函数
1. java类，继承UDF
2. 将jar包添加到hive的classpath

        hive>add JAR /home/hadoop/udf.jar;

3. 临时函数与开发好的java class关联
    
        Hive>create temporary function toprovince as 'cn.udf.ToProvince';

### transform

    TRANSFORM (xxx,xxx) USING 'python xxx.py' AS (xxx,xxx,xxx)

脚本是我们自定义的函数
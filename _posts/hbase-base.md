---
title: hbase 初探
date: 2017-01-23 13:21:08
tags:
    - hbase
    - 大数据
---
### hbase 介绍
hbase 是一个分布式的数据库，基于hdfs实现。能够线性扩展。相比于传统的关系型数据库，他天生就具备分布式的一些优点，能够处理存储处理大数据量任务，数据安全备份，集群水平扩展。

hbase 是基于列存储的数据库。

### 安装
1. 解压，添加环境变量
2. 修改 hbase-site.xml

        <configuration>
            <property>
                <name>hbase.master</name>
                <value>acyouzi01:60000</value>
            </property>
            <property>
                <name>hbase.master.maxclockskew</name>
                <value>180000</value>
            </property>
            <property>
                <name>hbase.rootdir</name>
                <value>hdfs://acyouzi01:9000/hbase</value>
            </property>
            <property>
                <name>hbase.cluster.distributed</name>
                <value>true</value>
            </property>
            <property>
                <name>hbase.zookeeper.quorum</name>
                <value>acyouzi01</value>
            </property>
            <property>
                <name>hbase.zookeeper.property.dataDir</name>
                <value>/root/test/hbase</value>
            </property>
        </configuration>
    
    hbase.rootdir 配置的是 hdfs 的地址
    hbase.master.maxclockskew 文档上没查到啊？
    hbase.zookeeper.quorum 是 zookeeper 地址
    
3. 修改 conf/regionservers 配置在哪些节点上启动 region
    
        acyouzi01
        acyouzi02

4. 启动 

        // 启动 hbase 集群
        start-hbase.sh
    
        // 进入 hbase shell
        hbase shell
        
### 让 hbase 了解 hdfs 的配置
上面实际上并没有用到 hdfs 的 ha 机制，因为我的 hdfs 集群就是非 ha 的。如果想使用 hdfs ha 集群，或者你的一些关于 hdfs 的配置需要让 hbase 知道可以通过如下方式实现：
1. HADOOP_CONF_DIR 加入到 hbase-env.sh 
2. 软连接或直接 copy  hdfs-site.xml (or hadoop-site.xml) 到 hbase/conf/ 目录


### 一些日志配置
    
    hbase.tmp.dir 临时文件存放在本机的什么地方
    hbase.rootdir  在 hdfs 上的根地址
    hbase.cluster.distributed true|false 是否使用集群模式
    hbase.zookeeper.quorum  xx,xxx,xxxx
    
    # 这是一个可以优化的地方
    zookeeper.session.timeout 连接超时时间，默认三分钟。在内网里边其实1分钟就ok
    
    # master 
    hbase.master.port    master 服务的端口号
    hbase.master.info.port 16010   master 图形界面的端口号
    hbase.master.info.bindAddress 
    
    # 日志清理
    hbase.master.logcleaner.plugins
    hbase.master.logcleaner.ttl
    
    # regionserver
    hbase.regionserver.port 16020 默认
    hbase.regionserver.info.port 16030 默认
    hbase.regionserver.info.bindAddress
    
    # RPC Listener 实例的数量
    hbase.regionserver.handler.count  30
    
    # 传输相关
    hbase.ipc.server.callqueue.handler.factor
    hbase.ipc.server.callqueue.read.ratio
    hbase.ipc.server.callqueue.scan.ratio
    hbase.regionserver.msginterval  
    hbase.ipc.client.tcpnodelay true|false
    hbase.rpc.timeout
    
### hbase 基本概念
1. rowkey

> 数据会按照行键的字典顺序依次排列，设计行键是要充分利用这一特性，把经常一起读的放到一起

2. Columns Family

> 是表 schema 的一部分，在表定义时定义

3. Cell

> 通过 {row key, columnFamily, version} 确定的一个存储单元，不区分数据类型，字节码形式存储。


### 命令使用
1. namespace

        namespace是表的逻辑分组，hbase 默认两个namespace： hbase, default
        create_namespace 'xx'
        drop_namespace 'xx'
        describe_namespace 'xx'
        list_namespace

2. 创建表

        create '表名', '列族名1','列族名2','列族名N'
        create 'namespace:表名', '列族名1','列族名2','列族名N'
        
        list 查看所有表
        list 'ns:abc.*'
        exists '表名' 
        is_enabled '表名'
        is_disabled ‘表名’
        disable 't2'
        enable 't2'

3. 添加数据
    
        put  ‘表名’, ‘rowKey’, ‘列族 : 列‘  ,  '值'
        put 'ns1:t1', 'r1', 'c1', 'value'
        put 't1', 'r1', 'c1', 'value', ts1   ts1 是 timestamp
        put 't1', 'r1', 'c1', 'value', {ATTRIBUTES=>{'mykey'=>'myvalue'}}
        put 't1', 'r1', 'c1', 'value', ts1, {ATTRIBUTES=>{'mykey'=>'myvalue'}}

4. get 数据
    
        get '表名','rowkey','列族：列’
        get 'ns1:t1', 'r1'
        get 't1', 'r1', {TIMERANGE => [ts1, ts2]}
        get 't1', 'r1', {COLUMN => 'c1'}
        get 't1', 'r1', {COLUMN => 'c1', ATTRIBUTES => {'mykey'=>'myvalue'}}

5. scan 

        scan "表名"
        scan "表名" , {COLUMNS=>'列族名:列名'}
        scan 'ns1:t1', {COLUMNS => ['c1', 'c2'], LIMIT => 10, STARTROW => 'xyz'}
        scan 't1', {COLUMNS => ['c1', 'c2'], LIMIT => 10, STARTROW => 'xyz'}
        scan 't1', {COLUMNS => 'c1', TIMERANGE => [1303668804, 1303668904]}

6. 删除清空

        alter 't1',{ NAME => 'f3', METHOD => 'delete'} 删除列簇
        delete  ‘表名’ ,‘行名’ , ‘列族：列'
        deleteall '表名','rowkey' 删除行
        truncate '表名'  清空表
        drop 't1' 删除表，需要先禁用再删除

7. 其他语句
    
        count  '表名' 统计行数

8. 过滤器
    
        等值过滤器
        scan 't1', FILTER=>"ValueFilter(=,'binary:xx')"
        
        列前缀过滤器
        scan 't1', FILTER=>"ColumnPrefixFilter('xx')"
        
        可以进行关系运算
        scan 't1', FILTER=>"ColumnPrefixFilter('xxx') AND ValueFilter ValueFilter(=,'substring:zzz')"
        
        只取出 key
        scan 't1', FILTER=>"KeyOnlyFilter()"
        
        前缀过滤器，过滤 rowkey 的前缀
        scan 't1', FILTER=>"PrefixFilter('zz')"

### 架构原理

![image](hbase-struct.png)

HBase 由 HMaster 和 HRegionServer 组成。

HBase 中的每张表都通过行键按照一定的范围被分割成多个子表（HRegion），默认一个 HRegion 超过 256M 就要被分割成两个。

HRegion 由 HRegionServer 管理，管理哪些 HRegion 由 HMaster 分配。

HRegion 中每个列族 (Column Family) 创建一个 Store 实例，一个 HRegion 有多少个列族就有多少个 Store。

每个 Store 都会有 0 个或多个 StoreFile 与之对应。每个 StoreFile 都会对应一个 HFile，HFile 就是实际的存储文件

每个 Store 还拥有一个 MemStore 内存缓存实例。

HFile：HBase 中 KeyValue 数据的存储格式，HFile 是 Hadoop 的二进制格式文件

MemStore：MemStore 即内存里放着的保存 KEY/VALUE 映射的 MAP，当 MemStore（默认 64MB）写满之后，会开始 flush 到hdfs

HLog File：HBase 中 WAL（Write Ahead Log）的存储格式，物理上是 Hadoop 的 Sequence File

### meta 表结构
通过 scan 'hbase:meta' 可以查看 meta 表中存储的数据，分析可以得出：
1. rowkey = tablename,region的StartKey,timestamp
2. 有一个列簇 info ，其中 regioninfo 存储了 NAME STARTKEY ENDKEY 等信息，server 列存储了 主机名 ip 等信息

### 写数据流程
1. 定位 region 信息，先通过 zookeeper 获取 meta 表的 region 信息，然后通过 meta 表找到 HRegionServer
2. RPC 连接 HRegionServer 发送写请求。
3. HRegionServer 首先把请求写到 HLog 中，然后在写入 MemStore。
4. 返回客户端写入成功
5. 当 MemStore 上的数据丢失可以从 HLog 恢复，当 MemStore 数据达到大小会刷写到 HDFS
6. 当有多个 StoreFile file 时会触发 Compact 操作，合并为一个。
7. 当 StoreFile 超过一定大小会触发 Split 操作， 相当于把一个大的 region 分成两个小的，或许还会调度两个 region 到不同机器

#### 刷写相关参数

    hbase.hregion.memstore.flush.size   default:134217728   128M
    hbase.hregion.memstore.block.multiplier 4  当一个region 的 memstore 总大小超过 hbase.hregion.memstore.block.multiplier * hbase.hregion.memstore.flush.size 时触发，然后会阻止 region 的写操作，强制刷写到memstore hdfs.
    
    总内存使用量超过这个值就刷写
    hbase.regionserver.global.memstore.size  = hbase.regionserver.global.memstore.upperLimit(0.4) * heapSize
    总内存使用量低于这个值停止刷写
    hbase.regionserver.global.memstore.size.lower.limit  = hbase.regionserver.global.memstore.lowerLimit * heapSize

    hbase.regionserver.max.logs HLog 数量达到上限，触发刷写
    
    # 较少的io线程，适合处理单次请求较大的put
    # 较多的io线程，适合单次请求较小，请求频率较高的场景。
    hbase.regionserver.handler.count 10
    
#### Compact 相关参数

    hbase.hstore.compaction.min 3 至少三个满足条件的 storefile 合并才会启动
    hbase.hstore.compaction.max 10 一次合并最多选择 10 storefile
    hbase.hstore.compaction.min.size 128M storefile小于该值的总是会被合并
    hbase.hstore.compaction.max.size  storefile大于该值的总是不会被合并
    hbase.hstore.compaction.ratio 

### Split 相关过程
1. HRegionServer 在 /hbase/region-in-transition 目录下创建 region-name 节点，状态为 SPLITTING.
2. master watch 了 /hbase/region-in-transition 节点，所以能知道节点的变化
3. region server 标记 region 下线，刷写缓存，这个时候客户端不能访问这个 region
4. region server 分裂 region 
5. 更新 meta 表中的信息。
6. region server 更新 znode 状态为 SPLIT.
7. master 知道分裂完成，判断是否调度 region 到其他 region server
    
#### Split 相关参数

    hbase.hregion.max.filesize default:10737418240 10G
    
### 读数据相关流程
1. 根据rowkey查询META表，获取所在region信息
2. 连接regionServer查询数据，先查询memStore，有则返回，无则继续下一步
3. 查找读缓存BlockCache是有rowkey的对应数据，有就返回，没有就继续查询
4. 到 storefile 中查找读 block 数据，可能要读多个 block ，读取的 block 数据会被放到 BlockCache 中。如果没找到直接返回 null

#### 相关参数
    
    # storefile 读缓存占内存的百分比。
    hfile.block.cache.size  

### HBase 的一些思考
rowkey 应该写入尽量分散，但是实际上很多时候我们会考虑设计 rowkey 使相近的，或者经常一起读取的内容存储在一起，这两者并不冲突。考虑如果向hbase中存储订信息，我们希望不同用户的信息尽量平均分布在不同的region sever 中，相同用户的订单则应当存储在一起。可以把rowkey的前半部分设为用户信息的hash，后半部分设为订单信息的编号，这样的存储场景还有很多。

考虑 hbase 可以与关系型数据库或者缓存数据库组合使用。缓存中存放一些经常需要读取的数据，如订单列表。在hbase中存放订单详情。当客户对内容更新时，考虑 hbase 写入较慢，先更新缓存，在更新缓存之前，先写入一个更新hbase的消息到消息队列，异步更 新 hbase.保证消息最终一致。如果这中间发生宕机，可以通过消息队列中的消息把 hbase 和缓存中的数据同步。(只是一个想法，不知道行不行)

列簇应该少一点 1-2个

rowkey 应该短一点。

根据hbase 的集群架构来看，region server 还是点单。
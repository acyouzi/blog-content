---
title: flume 全接触
date: 2017-01-23 13:33:28
tags:
    - flume
    - 大数据
---
### flume 是什么
flume 是分布式，可靠，高可用的日志采集传输系统。

可以采集文件，socket数据包等各种形式的数据源。

同时可以把采集到的数据下沉到 hdfs,hive,hbase,kafka等外部存储系统。

而且其配置简单，适用于各种日志采集场景。

### flume 的相关概念
1. agent 是一个采集传输数据的数据传递员角色。其内部有 source,channel,sink 三个部分组成

2. source 数据的来源

3. channel 数据内部的传输通道，有缓存功能

4. sink 下沉地，数据传输的目的地 可以是下一级 agent 或者存储系统

![image](flume-struct.png)

### 使用
1. 修改 flume-env.sh 里面的 JAVA_HOME
2. 编写配置文件 flume-simple.conf
        
        # 官网给出的简单配置文件
        # a1 是代理的名字
        # r1 是 sources 的名字，
        # k1 是 sink 的名字
        # c1 是 channel 的名字
        a1.sources = r1
        a1.sinks = k1
        a1.channels = c1
        
        # source r1 是一个 netcat 类型的源，也就是大名鼎鼎的 nc
        a1.sources.r1.type = netcat
        a1.sources.r1.bind = localhost
        a1.sources.r1.port = 44444
        
        # sink 类型是 logger ,flume 使用 log4j 输出日志
        # 关于具体的日志输出可以在 conf/log4j.properties 中配置
        a1.sinks.k1.type = logger
        
        # channel 是
        a1.channels.c1.type = memory
        a1.channels.c1.capacity = 1000
        a1.channels.c1.transactionCapacity = 100
        
        # Bind the source and sink to the channel
        a1.sources.r1.channels = c1
        a1.sinks.k1.channel = c1
    
3. 修改 conf/log4j.properties,下面是默认的 log4j 配置文件

        # 首先是几个自定义的参数，也可以做在 jvm 启动时通过 -Dxxx 指定
        # -Dflume.root.logger=DEBUG,console when launching flume.
        # flume.root.logger 的值最后会赋值给 rootLogger 
        flume.root.logger=INFO,LOGFILE
        flume.log.dir=./logs
        flume.log.file=flume.log
        
        log4j.logger.org.apache.flume.lifecycle = INFO
        log4j.logger.org.jboss = WARN
        log4j.logger.org.mortbay = INFO
        log4j.logger.org.apache.avro.ipc.NettyTransceiver = WARN
        log4j.logger.org.apache.hadoop = INFO
        log4j.logger.org.apache.hadoop.hive = ERROR
        
        # rootLogger 的语法为log4j.rootLogger = [ level ] , appenderName, appenderName
        # [ level ] 是日志级别，一般只使用 ERROR、WARN、INFO、DEBUG 四个级别
        # appenderName 指定输出到哪里，可以指定多个
        # 是通过 log4j.appender.xxxx = yyyy 定义的
        log4j.rootLogger=${flume.root.logger}
        
        # 输出到文件
        log4j.appender.LOGFILE=org.apache.log4j.RollingFileAppender
        log4j.appender.LOGFILE.MaxFileSize=100MB
        log4j.appender.LOGFILE.MaxBackupIndex=10
        log4j.appender.LOGFILE.File=${flume.log.dir}/${flume.log.file}
        log4j.appender.LOGFILE.layout=org.apache.log4j.PatternLayout
        log4j.appender.LOGFILE.layout.ConversionPattern=%d{dd MMM yyyy HH:mm:ss,SSS} %-5p [%t] (%C.%M:%L) %x - %m%n
        
        # 
        log4j.appender.DAILY=org.apache.log4j.rolling.RollingFileAppender
        log4j.appender.DAILY.rollingPolicy=org.apache.log4j.rolling.TimeBasedRollingPolicy
        log4j.appender.DAILY.rollingPolicy.ActiveFileName=${flume.log.dir}/${flume.log.file}
        log4j.appender.DAILY.rollingPolicy.FileNamePattern=${flume.log.dir}/${flume.log.file}.%d{yyyy-MM-dd}
        log4j.appender.DAILY.layout=org.apache.log4j.PatternLayout
        log4j.appender.DAILY.layout.ConversionPattern=%d{dd MMM yyyy HH:mm:ss,SSS} %-5p [%t] (%C.%M:%L) %x - %m%n
        
        # console
        log4j.appender.console=org.apache.log4j.ConsoleAppender
        log4j.appender.console.target=System.err
        log4j.appender.console.layout=org.apache.log4j.PatternLayout
        log4j.appender.console.layout.ConversionPattern=%d (%t) [%p - %l] %m%n

4. 启动

        flume-ng agent --conf conf --conf-file ./conf/flume-simple.conf --name a1 -Dflume.root.logger=DEBUG,console -Dorg.apache.flume.log.printconfig=true -Dorg.apache.flume.log.rawdata=true
        
        --name 一定要与配置文件中的相同
        
### flume 从zookeeper上加载配置文件
这是一个还在试验的特性，通过命令行命令使用

    flume-ng agent –conf conf -z zkhost:2181,zkhost1:2181 -p /flume –name a1 -Dflume.root.logger=INFO,console


### flume 可以把一个数据源输出到多个 channel 和 sink

![image](flume-multi-flow.png)

多个 flume 的配置

    # list the sources, sinks and channels in the agent
    agent_foo.sources = avro-AppSrv-source1 exec-tail-source2
    agent_foo.sinks = hdfs-Cluster1-sink1 avro-forward-sink2
    agent_foo.channels = mem-channel-1 file-channel-2
    
    # flow #1 configuration
    agent_foo.sources.avro-AppSrv-source1.channels = mem-channel-1
    agent_foo.sinks.hdfs-Cluster1-sink1.channel = mem-channel-1
    
    # flow #2 configuration
    agent_foo.sources.exec-tail-source2.channels = file-channel-2
    agent_foo.sinks.avro-forward-sink2.channel = file-channel-2

### flume可以通过下面几种机制收集日志

1. Avro
    
        a1.sources.r1.type = avro
        a1.sources.r1.bind = 0.0.0.0
        a1.sources.r1.port = 4141
    
2. Thrift
        
        a1.sources.r1.type = thrift
        a1.sources.r1.bind = 0.0.0.0
        a1.sources.r1.port = 4141

3. Exec
执行一段脚本收集日志，容易丢失数据
    
        a1.sources.r1.type = exec
        a1.sources.r1.command = tail -F /var/log/secure
    
        // or 指定使用的 shell 
        a1.sources.tailsource-1.type = exec
        a1.sources.tailsource-1.shell = /bin/bash -c
        a1.sources.tailsource-1.command = for i in /path/*.txt; do cat $i; done

4. Spooling Directory Source
监视一个文件夹，当有新文件时进行采集，可靠，不容易丢失数据，如果服务停止，会记录发送到哪里，然后重启时继续发送。

        a1.sources.src-1.type = spooldir
        a1.sources.src-1.spoolDir = /var/log/apache/flumeSpool
        a1.sources.src-1.fileHeader = true
        
        // 还可以设置 
        // deletePolicy  true 是不是在消费完删除日志文件
        // fileHeader 是不是在文件头中保存文件路径
        // includePattern 正则表达式
        // ignorePattern 正则表达式，忽略文件
        // consumeOrder oldest|youngest|random 消费规则
        // pollDelay 多长时间轮训一次文件夹
        // recursiveDirectorySearch false 是否监视子文件夹
        // maxBackoff 如果 channel 满了，等待的最大时间
        // batchSize 传输的文件块大小
        // inputCharset UTF-8 
        
5. Taildir Source
这是一个正在试验的功能

6. Kafka Source

        tier1.sources.source1.type = org.apache.flume.source.kafka.KafkaSource
        tier1.sources.source1.batchSize = 5000  // 每次写入 channel 的数据条数
        tier1.sources.source1.batchDurationMillis = 2000 // 每次写入channel 的间隔时间
        tier1.sources.source1.kafka.bootstrap.servers = localhost:9092
        tier1.sources.source1.kafka.topics = test1, test2
        tier1.sources.source1.kafka.consumer.group.id = custom.g.id  //消费者分组
        
        // 或者使用正则表达式订阅话题
        tier1.sources.source1.kafka.topics.regex = ^topic[0-9]$
        
7. NetCat Source

        a1.sources.r1.type = netcat
        a1.sources.r1.bind = 0.0.0.0
        a1.sources.r1.port = 6666
        
8. 其他 source 

        Syslog TCP Source
        Multiport Syslog TCP Source
        Syslog UDP Source
        HTTP Source
        Stress Source
        Scribe Source 
        
### flume sink 类型

1. HDFS Sink

        a1.sinks.k1.type = hdfs
        a1.sinks.k1.hdfs.path = hdfs://host/flume/events/%y-%m-%d/%H%M/%S
        a1.sinks.k1.hdfs.filePrefix = events-
        // 读时间进行舍入，会影响 %t 以外的所有转义符
        a1.sinks.k1.hdfs.round = true
        a1.sinks.k1.hdfs.roundValue = 10
        a1.sinks.k1.hdfs.roundUnit = minute
        
   [参看这里能用的转义符](http://flume.apache.org/FlumeUserGuide.html#hdfs-sink)
    
        hdfs.filePrefix	FlumeData  // 创建的文件前缀
        hdfs.fileSuffix  // 后缀
        hdfs.rollInterval // 多长时间滚动一次 
        hdfs.rollSize  // 多少字节滚动一次
        hdfs.rollCount  //  多少次写入，滚动一次
        hdfs.idleTimeout	0	// 空闲多长时间后关闭文件，0是不关闭
        hdfs.fileType	SequenceFile(default)|DataStream|CompressedStream
        hdfs.round	false 
        hdfs.roundValue	1	
        hdfs.roundUnit
        serializer	TEXT

2. Hive Sink

        a1.sinks.k1.type = hive
        a1.sinks.k1.hive.metastore = thrift://127.0.0.1:9083
        a1.sinks.k1.hive.database = logsdb
        a1.sinks.k1.hive.table = weblogs
        a1.sinks.k1.hive.partition = asia,%{country},%y-%m-%d-%H-%M
        a1.sinks.k1.useLocalTimeStamp = false
        a1.sinks.k1.round = true
        a1.sinks.k1.roundValue = 10
        a1.sinks.k1.roundUnit = minute
        a1.sinks.k1.serializer = DELIMITED
        a1.sinks.k1.serializer.delimiter = "\t"
        a1.sinks.k1.serializer.serdeSeparator = '\t'
        a1.sinks.k1.serializer.fieldnames =id,,msg

3. Logger Sink
        
        a1.sinks.k1.type = logger

4. Avro Sink

        a1.sinks.k1.type = avro
        a1.sinks.k1.hostname = 10.10.10.10
        a1.sinks.k1.port = 4545

5. Thrift Sink

        a1.sinks.k1.type = thrift
        a1.sinks.k1.hostname = 10.10.10.10
        a1.sinks.k1.port = 4545

6. File Roll Sink
 
        a1.sinks.k1.type = file_roll
        a1.sinks.k1.sink.directory = /var/log/flume

7. HBaseSink
    
        a1.sinks.k1.type = hbase
        a1.sinks.k1.table = foo_table
        a1.sinks.k1.columnFamily = bar_cf
        a1.sinks.k1.serializer = org.apache.flume.sink.hbase.RegexHbaseEventSerializer
    
8. AsyncHBaseSink

        a1.sinks.k1.type = asynchbase
        a1.sinks.k1.table = foo_table
        a1.sinks.k1.columnFamily = bar_cf
        a1.sinks.k1.serializer = org.apache.flume.sink.hbase.SimpleAsyncHbaseEventSerializer

9. Kafka Sink
    
        a1.sinks.k1.type = org.apache.flume.sink.kafka.KafkaSink
        a1.sinks.k1.kafka.topic = mytopic
        a1.sinks.k1.kafka.bootstrap.servers = localhost:9092
        a1.sinks.k1.kafka.flumeBatchSize = 20
        a1.sinks.k1.kafka.producer.acks = 1
        a1.sinks.k1.kafka.producer.linger.ms = 1
        a1.sinks.ki.kafka.producer.compression.type = snappy

### channel
1. Memory Channel
        
        a1.channels.c1.type = memory
        a1.channels.c1.capacity = 10000
        a1.channels.c1.transactionCapacity = 10000
        a1.channels.c1.byteCapacityBufferPercentage = 20
        a1.channels.c1.byteCapacity = 800000

2. JDBC Channel，好像不能连接 mysql
3. Kafka Channel
        
        a1.channels.channel1.type = org.apache.flume.channel.kafka.KafkaChannel
        a1.channels.channel1.kafka.bootstrap.servers = kafka-1:9092,kafka-2:9092,kafka-3:9092
        a1.channels.channel1.kafka.topic = channel1
        a1.channels.channel1.kafka.consumer.group.id = flume-consumer

4. File Channel

        a1.channels.c1.type = file
        // 存储检查点
        a1.channels.c1.checkpointDir = /mnt/flume/checkpoint
        a1.channels.c1.dataDirs = /mnt/flume/data

### Channel Selectors
1. Replicating Channel Selector

        a1.channels = c1 c2 c3
        a1.sources.r1.selector.type = replicating
        a1.sources.r1.channels = c1 c2 c3
        a1.sources.r1.selector.optional = c3
    
    c1,c2 写入失败会导致错误，c3 是复制通道，失败会忽略

2. Multiplexing Channel Selector
    根据头不同进行通道选择

        a1.channels = c1 c2 c3 c4
        a1.sources.r1.selector.type = multiplexing
        a1.sources.r1.selector.header = state
        a1.sources.r1.selector.mapping.CZ = c1
        a1.sources.r1.selector.mapping.US = c2 c3
        a1.sources.r1.selector.default = c4

### Sink Processors
我们使用 Sink groups 组合一组 sink, 然后可以指定 processor 进行不同策略的调度
    
    // 定义组
    a1.sinkgroups = g1
    a1.sinkgroups.g1.sinks = k1 k2
    // 调度策略
    a1.sinkgroups.g1.processor.type = load_balance

### Load balancing Sink Processor

    processor.sinks
    processor.type load_balance
    processor.backoff	false 
    // 调度机制
    processor.selector	round_robin|random|FQCN 
    processor.selector.maxTimeOut	30000 



### 例子
node2 采集数据传递给 node1 , 然后 node1 下沉到 hdfs
描述

    node1:
        name:a1
        source: r1 type avro
        sinks:
            k1 type logger
            k2 type hdfs
        channels:
            c1 type memory
            c2 type memory
    node2:
        name:a1
        source: r1 type spooldir
        sink: k1 type memory
        channel: c1 type avro
    
node1 配置文件

    // node1
    # Name the components on this agent
    a1.sources = r1
    a1.sinks = k1 k2
    a1.channels = c1 c2
    
    # sources
    a1.sources.r1.type = avro
    a1.sources.r1.bind = 0.0.0.0
    a1.sources.r1.port = 4141
    a1.sources.r1.channels = c1 c2
    
    # Describe the sink
    a1.sinks.k1.type = logger
    a1.sinks.k1.channel = c1
    
    a1.sinks.k2.type = hdfs
    a1.sinks.k2.hdfs.path = hdfs://acyouzi01:9000/flume/events/%y
    a1.sinks.k2.hdfs.filePrefix = events-
    a1.sinks.k2.channel = c2
    
    # For all of the time related escape sequences, a header with the key “timestamp” must exist among the headers of the event (unless hdfs.useLocalTimeStamp is set to true)
    a1.sinks.k2.hdfs.useLocalTimeStamp = true
    
    # Use a channel which buffers events in memory
    a1.channels.c1.type = memory
    a1.channels.c1.capacity = 1000
    a1.channels.c1.transactionCapacity = 100
    
    a1.channels.c2.type = memory
    a1.channels.c2.capacity = 1000
    a1.channels.c2.transactionCapacity = 100

node2 配置文件

    // node2
    # Name the components on this agent
    a1.sources = r1
    a1.sinks = k1
    a1.channels = c1
    
    # sources
    a1.sources.r1.type = spooldir
    a1.sources.r1.spoolDir = /root/test/
    a1.sources.r1.fileHeader = true
    a1.sources.r1.channels = c1
    
    # Describe the sink
    a1.sinks.k1.type = avro
    a1.sinks.k1.hostname = acyouzi01
    a1.sinks.k1.port = 4141
    a1.sinks.k1.channel = c1
    
    # Use a channel which buffers events in memory
    a1.channels.c1.type = memory
    a1.channels.c1.capacity = 1000
    a1.channels.c1.transactionCapacity = 100
    
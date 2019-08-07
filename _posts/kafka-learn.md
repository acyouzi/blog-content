---
title: kafka 相关概念和使用
date: 2017-02-06 22:23:11
description: 
tags:
    - java
---
# 介绍
kafka 是一个开源消息系统，使用 scala 编写。类似于 JMS , 但不是 JMS 的规范实现。

## 相关概念
1. Producer
2. Consumer
3. Topic 
    
    一个特定的主题，或者理解为存储相同消息类型的一个或多个队列

4. Consumer Group
    
    一个 topic 可以有多个 group, 每个 group 可以有一到多个 consumer. topic 的消息在逻辑上会复制到多个 consumer group 上。如果一个 group 有多个 consumer. 每条消息只会被其中一个 consumer 消费。如果一个 topic 有多个 group, 每个 group 都会得到相同的消息。

5. Broker 一台 kafka 服务器。
6. Partition
    
    为了实现扩展性，一个非常大的topic可以分布到多个broker上，一个topic可以分为多个partition，每个partition是一个有序的队列。kafka只保证一个partition中的消息顺序发给consumer，不保证多个partition消息有序。

## consumer / partition / topic 的关系
1. 每个group中可以有多个consumer, 每个consumer属于一个consumer group
2. 每个 topic 中的一条消息只会被 consumer group 中的一个消费者消费。
3. 一个 topic 可以有多个 partition . 实际消费时是一个 partition 对应一个 consumer. 
4. 一个 partition 对应一个 consumer . 但是一个 consumer 可以消费一个 topic 的多个 partition.
5. 同一个group中不能有多于partitions个数的consumer同时消费，否则某些consumer将无法得到消息.

# 安装与使用
1. 解压，配置环境变量
2. 修改配置文件
    
        # 这个在集群中的每个节点都不能重复。
        broker.id=0
    
        # zookeeper 的地址也要修改
        zookeeper.connect=192.168.192.132:2181

3. 启动 zookeeper 
    
        # 可以启动自己配置的 zookeeper 也可以使用如下命令启动 kafka 自带的 zookeeper
        bin/zookeeper-server-start.sh config/zookeeper.properties

4. 启动 kafka 
    
        # 在每台机器上执行
        bin/kafka-server-start.sh [-daemon] config/server.properties

## 命令行使用 kafka 
1. 创建 topic 

        bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
        
2. 列出 topic 
    
        bin/kafka-topics.sh --list --zookeeper localhost:2181
    
3. 查看 topic 详细信息
        
        bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
    
4. 通过命令行生产消息    

        bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
    
5. 消费消息
    
        bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
    
6. 删除 topic 

        bin/kafka-topics.sh --delete --zookeeper localhost:2181 --topic test

# Paritition机制
Producer发送消息到broker时，会根据Paritition机制选择将其存储到哪一个Partition。如果Partition机制设置合理，所有消息可以均匀分布到不同的Partition里，这样就实现了负载均衡

Paritition机制可以通过指定Producer的paritition. class这一参数来指定，该class必须实现kafka.producer.Partitioner接口

默认算法是 hashcode % partitions，也可以自己实现。

# 消息应答机制
通过参数 request.required.acks 可以设置 producer 接收消息ack 的机制。默认为0.  
    
    # 0: producer不会等待broker发送ack  
    # 1: 当leader接收到消息之后发送ack  
    # 2: 当所有的follower都同步消息成功后发送ack.  
    acks=0 

# Consumer的负载均衡
当一个group中consumer加入或者离开时,会触发负载均衡。

    # 计算公式
    # 可能出现 consumer 空闲的情况
    每个 consumer 分得的 partition = ceil(partitions.len/consumers.len)

# 文件存储
在 kafka 数据存储目录下的 partition 以 topic + partition_id 命名。在 partition 文件夹下有一个 xxxx.index 和 xxxx.log 两种类型的文件。这些文件就是 Segment file

每个 partition 会被分成多个 Segment，默认会保留7天的 segment.

segment文件名为上一个segment文件最后一条消息的offset值. 索引文件存储元数据，索引文件中元数据指向对应数据文件中message的物理偏移地址. 

# 配置
## Boker 配置参数
Property | Description
---|---
broker.id | 每个 broker 唯一的整数ID
zookeeper.connect | ip:port
listeners=PLAINTEXT://host:9092 | 监听地址
advertised.listeners | 
advertised.port | 
num.partitions | topic的默认partitions数目
auto.create.topics.enable = true |
delete.topic.enable = false | 
log.dirs | 数据存放目录
log.flush.interval.messages |
log.flush.interval.ms | 
log.retention.bytes | 保留的日志大小
log.retention.hours | 保留的日志时间
log.roll.hours | (默认 7 * 24 ) 强制Kafka在达到该时间后生成一个新的log文件，而不管log.segment.bytes是否达到指定的大小
log.segment.bytes | topic的partition存储是以目录中的segment files存在的，该值控制segment file的最大大小，如果文件达到该值后，会重新生成一个新的日志文件
log.cleanup.policy | delete/compact
log.retention.check.interval.ms=60000 | 日志片段文件的检查周期，查看它们是否达到了删除策略的设置（log.retention.hours或log.retention.bytes）
log.cleaner.enable=false | 是否开启压缩
log.cleaner.delete.retention.ms = 1 day | 对于压缩的日志保留的最长时间
message.max.bytes | 消息体最大值
num.io.threads | 用来执行 io 请求的线程数，应等于磁盘数
queued.max.requests | 在I/O线程队列中的请求达到多少多少个之后，network线程停止接收新的请求
socket.send.buffer.bytes | 发送缓冲
socket.receive.buffer.bytes	| socket的接收缓冲区，SO_RCVBUFF
replica.lag.max.messages = 4000 | 如果relicas落后太多,将会认为此partition relicas已经失效。而一般情况下,因为网络延迟等原因,总会导致replicas中消息同步滞后。如果消息严重滞后,leader将认为此relicas网络延迟较大或者消息吞吐能力有限。在broker数量较少,或者网络不足的环境中,建议提高此值.
replica.socket.timeout.ms= 30 * 1000 | leader与relicas的socket超时时间
replica.socket.receive.buffer.bytes=64 * 1024 | leader复制的socket缓存大小
replica.fetch.max.bytes = 1024 * 1024 | replicas每次获取数据的最大字节数
replica.fetch.wait.max.ms = 500 | replicas同leader之间通信的最大等待时间，失败了会重试
replica.fetch.min.bytes =1 | 每一个fetch操作的最小数据尺寸,如果leader中尚未同步的数据不足此值,将会等待直到数据达到这个大小
num.replica.fetchers = 1 | leader中进行复制的线程数，增大这个数值会增加relipca的IO

## Producer 配置参数
Property | Description
---|---
acks = 0 | 0,1,-1
bootstrap.servers | host:port,...
key.serializer | key的序列化方式，若是没有设置，同serializer.class
value.serializer | 
buffer.memory | 
retries | 重试次数
batch.size = 16384 | 聚合多长请求到一定大小然后一起发送
linger.ms | batch 延迟的上限
client.id | string
connections.max.idle.ms = 540000 | 多长时间以后关闭空闲连接
client.id="" | 用户随意指定，但是不能重复，主要用于跟踪记录消息
partitioner.class=org.apache.kafka.clients.producer.internals.DefaultPartitioner | 分区的策略，默认是取模
request.timeout.ms = 10000 | 消息发送的最长等待时间
send.buffer.bytes=100*1024 | socket的缓存大小
retry.backoff.ms = 100 | 每次失败后的间隔时间
topic.metadata.refresh.interval.ms = 600 * 1000 | 生产者定时更新topic元信息的时间间隔 ，若是设置为0，那么会在每个消息发送后都去更新数据
queue.buffering.max.ms = 5000 | 异步模式下缓冲数据的最大时间。例如设置为100则会集合100ms内的消息后发送，这样会提高吞吐量，但是会增加消息发送的延时
queue.buffering.max.messages = 10000 | 异步模式下缓冲的最大消息数，同上
queue.enqueue.timeout.ms = -1 | 异步模式下，消息进入队列的等待时间。若是设置为0，则消息不等待，如果进入不了队列，则直接被抛弃
batch.num.messages=200 | 

## Consumer 配置参数
Property | Description
---|---
client.id | 
bootstrap.servers | host:port,...
key.serializer | key 的序列化方式
value.serializer | value 的序列化方式
group.id | 决定该Consumer归属的唯一组ID
fetch.min.bytes = 1 | server发送到消费端的最小数据，若是不满足这个数值则会等待直到满足指定大小。默认为1表示立即接收。
heartbeat.interval.ms | 用于确认 consumer 活着，在 consumer group 协调的时候会用到，值等于 session.timeout.ms 的三分之一
session.timeout.ms = 10000 | 用于检测 consumer 是否发生故障
max.poll.interval.ms = 300000 | 两次 poll 之间的最大间隔
max.poll.records = 500 | 一次获取的最大记录跳数
request.timeout.ms = 305000 | 等待超时时间
retry.backoff.ms |
reconnect.backoff.ms | 
auto.offset.reset | earliest/latest/none(throw exception)
enable.auto.commit = true | 自动提交

# kafka java api
注意如果 bootstrap.servers 写成 ip 形式，最好在 config/server.properties 中把 listeners=PLAINTEXT://192.168.192.132:9092 或者 advertised.listeners=PLAINTEXT://192.168.192.132:9092 改成 ip 物理网卡的 ip 地址，否则可能会发生连不上 Broker 的情况。

## producer

    public class MyProducer {
        public static void main(String[] args) throws ExecutionException, InterruptedException {
            Properties props = new Properties();
            props.put("bootstrap.servers", "192.168.192.132:9092");
            props.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
            props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
            KafkaProducer<Integer, String> producer = new KafkaProducer<>(props);
            for (int i = 0; i < 100; i++) {
                producer.send(new ProducerRecord<Integer, String>("test",i,"test_"+i));
            }
            producer.close();
        }
    }

## consumer
参看 [http://kafka.apache.org/0101/javadoc/index.html?org/apache/kafka/clients/consumer/KafkaConsumer.html](http://kafka.apache.org/0101/javadoc/index.html?org/apache/kafka/clients/consumer/KafkaConsumer.html)

    public class MyConsumer {
        public static void main(String[] args) {
            Properties props = new Properties();
            props.put("bootstrap.servers", "192.168.192.132:9092");
            props.put("group.id", "test");
            props.put("enable.auto.commit", "true");
            props.put("auto.commit.interval.ms", "1000");
            props.put("key.deserializer", "org.apache.kafka.common.serialization.IntegerDeserializer");
            props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
            KafkaConsumer<Integer, String> consumer = new KafkaConsumer<>(props);
            consumer.subscribe(Arrays.asList("test"));
            while (true) {
                ConsumerRecords<Integer, String> records = consumer.poll(100);
                for (ConsumerRecord<Integer, String> record : records)
                    System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
            }
        }
    }
    
## stream 
有点类似 strom 的功能了，使 kafka 具有了流式处理的能力.

参看 [http://kafka.apache.org/0101/javadoc/index.html?org/apache/kafka/streams/KafkaStreams.html](http://kafka.apache.org/0101/javadoc/index.html?org/apache/kafka/streams/KafkaStreams.html)

# HA
参考：

[Kafka设计解析（二）- Kafka High Availability （上）](http://www.jasongj.com/2015/04/24/KafkaColumn2/)

[Kafka设计解析（三）- Kafka High Availability （下）](http://www.jasongj.com/2015/06/08/KafkaColumn3/)

[Kafka源码分析 KafkaController](http://zqhxuyuan.github.io/2016/02/23/2016-02-23-Kafka-Controller/)

## Leader 选举
// 待补充
## 备份策略
// 待补充
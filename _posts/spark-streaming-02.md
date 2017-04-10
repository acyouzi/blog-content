---
title: Spark源码-streaming-02-接收数据的可靠性
date: 2017-04-10 21:07:16
tags:
    - spark
---
## 总结
1. 一个疑惑：在接收的数据真正交给 blockmanager 之前是在 blockgenerate 中存储的，这里面并没有任何持久化保证，也就是说如果这个时候宕机是不是当前在 blockgenerate 中还没有交个 blockManager 的数据会全部丢失掉？
2. 默认情况下 spark streaming 使用 BlockManagerBasedBlockHandler 保存块，并且默认持久化策略是 MEMORY_AND_DISK_SER_2
3. WriteAheadLogBasedBlockHandler 默认使用 FileBasedWriteAheadLog 写日志，这种方式每个一段时间( spark.streaming.receiver.writeAheadLog.rollingIntervalSecs ) 滚动一次，生成一个新的日志文件，使得单个日志文件不会太大。
4. Kafka 的 DirectStream 另外一种数据可靠性的方法，这种方法跳出了正在 spark streaming 的模式，不会再单独启动 Receiver 接收数据
5. DirectStream 相当于是把 Kafka 看做类似 HDFS 一样的底层文件系统，由 Spark Streaming 来负责整个 offset 的侦测、batch 分配、实际读取数据，些分 batch 的信息都是 checkpoint 到了可靠存储中。

## reciver 预写日志
1. 如果我们在 spark stream 的配置文件中指定了 spark.streaming.receiver.writeAheadLog.enable 为 true ，则 ReceiverSupervisorImpl 实例化时得到的 receivedBlockHandler 实例是一个 WriteAheadLogBasedBlockHandler，还有一个需要注意的地方是不管创建的是 WriteAheadLogBasedBlockHandler 还是 BlockManagerBasedBlockHandler 都会传入一个 storageLevel ,这个参数配置的是底层 blockManager 的持久化级别，默认是 MEMORY_AND_DISK_SER_2 ，如果已经开启乐预写日志可以把持久化级别后面的 2 给去掉。

          private val receivedBlockHandler: ReceivedBlockHandler = {
            if (WriteAheadLogUtils.enableReceiverLog(env.conf)) {
              if (checkpointDirOption.isEmpty) {
                throw new SparkException(
                  "Cannot enable receiver write-ahead log without checkpoint directory set. " +
                    "Please use streamingContext.checkpoint() to set the checkpoint directory. " +
                    "See documentation for more details.")
              }
              new WriteAheadLogBasedBlockHandler(env.blockManager, env.serializerManager, receiver.streamId,
                receiver.storageLevel, env.conf, hadoopConf, checkpointDirOption.get)
            } else {
              new BlockManagerBasedBlockHandler(env.blockManager, receiver.storageLevel)
            }
          }

2. WriteAheadLogBasedBlockHandler 的构造函数中会实例化真正负责存储的 writeAheadLog， 具体实例化方法如下：

          private def createLog(
              isDriver: Boolean,
              sparkConf: SparkConf,
              fileWalLogDirectory: String,
              fileWalHadoopConf: Configuration
            ): WriteAheadLog = {
            
            // 配置的预写日志类
            val classNameOption = if (isDriver) {
              sparkConf.getOption(DRIVER_WAL_CLASS_CONF_KEY)
            } else {
              sparkConf.getOption(RECEIVER_WAL_CLASS_CONF_KEY)
            }
            val wal = classNameOption.map { className =>
              try {
                // 实例化用户配置的预写日志类
                instantiateClass(
                  Utils.classForName(className).asInstanceOf[Class[_ <: WriteAheadLog]], sparkConf)
              } catch {
                case NonFatal(e) =>
                  throw new SparkException(s"Could not create a write ahead log of class $className", e)
              }
            }.getOrElse {
              // 默认的预写日志类
              new FileBasedWriteAheadLog(sparkConf, fileWalLogDirectory, fileWalHadoopConf,
                getRollingIntervalSecs(sparkConf, isDriver), getMaxFailures(sparkConf, isDriver),
                shouldCloseFileAfterWrite(sparkConf, isDriver))
            }
            // 受参数 spark.streaming.driver.writeAheadLog.allowBatching 控制
            // 这个类另外开启了一个线程负责存储
            // 调用 write 方法时仅仅是把消息存储到一个队列里面
            // 真正存储时会对队列中所以的消息按时间排序，并聚合成一条消息
            // 然后使用前面创建的 wal 实例写入
            if (isBatchingEnabled(sparkConf, isDriver)) {
              new BatchedWriteAheadLog(wal, sparkConf)
            } else {
              wal
            }
          }
    
3. 下面看 FileBasedWriteAheadLog 类, 看这个类的 write 具体实现：

          def write(byteBuffer: ByteBuffer, time: Long): FileBasedWriteAheadLogSegment = synchronized {
            var fileSegment: FileBasedWriteAheadLogSegment = null
            var failures = 0
            var lastException: Exception = null
            var succeeded = false
            while (!succeeded && failures < maxFailures) {
              try {
                // 关键是这里，根据时间得到 LogWriter
                // 每个 LogWriter 对应一个日志文件，滚动日志
                fileSegment = getLogWriter(time).write(byteBuffer)
                if (closeFileAfterWrite) {
                  resetWriter()
                }
                succeeded = true
              } catch {
                case ex: Exception =>
                  lastException = ex
                  logWarning("Failed to write to write ahead log")
                  resetWriter()
                  failures += 1
              }
            }
            if (fileSegment == null) {
              logError(s"Failed to write to write ahead log after $failures failures")
              throw lastException
            }
            fileSegment
          }
          
    getLogWriter 方法， 多长时间滚动一次可以通过如下参数控制 spark.streaming.receiver.writeAheadLog.rollingIntervalSecs
          
          private def getLogWriter(currentTime: Long): FileBasedWriteAheadLogWriter = synchronized {
            if (currentLogWriter == null || currentTime > currentLogWriterStopTime) {
              resetWriter()
              currentLogPath.foreach {
                pastLogs += LogInfo(currentLogWriterStartTime, currentLogWriterStopTime, _)
              }
              currentLogWriterStartTime = currentTime
              currentLogWriterStopTime = currentTime + (rollingIntervalSecs * 1000)
              val newLogPath = new Path(logDirectory,
                timeToLogFile(currentLogWriterStartTime, currentLogWriterStopTime))
              currentLogPath = Some(newLogPath.toString)
              currentLogWriter = new FileBasedWriteAheadLogWriter(currentLogPath.get, hadoopConf)
            }
            currentLogWriter
          }
    
    然后 write 方法调用 hadoop API 向 hadoop 写数据

4. WriteAheadLogBasedBlockHandler#storeBlock 方法中同时向 blockmanager 还有 writeAheadLog 写日志，实现了 reciver 上的可靠性

          def storeBlock(blockId: StreamBlockId, block: ReceivedBlock): ReceivedBlockStoreResult = {
    
            var numRecords = Option.empty[Long]
            // Serialize the block so that it can be inserted into both
            val serializedBlock = block match {
              case ArrayBufferBlock(arrayBuffer) =>
                numRecords = Some(arrayBuffer.size.toLong)
                serializerManager.dataSerialize(blockId, arrayBuffer.iterator)
              case IteratorBlock(iterator) =>
                val countIterator = new CountingIterator(iterator)
                val serializedBlock = serializerManager.dataSerialize(blockId, countIterator)
                numRecords = countIterator.count
                serializedBlock
              case ByteBufferBlock(byteBuffer) =>
                new ChunkedByteBuffer(byteBuffer.duplicate())
              case _ =>
                throw new Exception(s"Could not push $blockId to block manager, unexpected block type")
            }
        
            // Store the block in block manager
            val storeInBlockManagerFuture = Future {
              val putSucceeded = blockManager.putBytes(
                blockId,
                serializedBlock,
                effectiveStorageLevel,
                tellMaster = true)
              if (!putSucceeded) {
                throw new SparkException(
                  s"Could not store $blockId to block manager with storage level $storageLevel")
              }
            }
        
            // Store the block in write ahead log
            val storeInWriteAheadLogFuture = Future {
              writeAheadLog.write(serializedBlock.toByteBuffer, clock.getTimeMillis())
            }
        
            // Combine the futures, wait for both to complete, and return the write ahead log record handle
            val combinedFuture = storeInBlockManagerFuture.zip(storeInWriteAheadLogFuture).map(_._2)
            val walRecordHandle = ThreadUtils.awaitResult(combinedFuture, blockStoreTimeout)
            WriteAheadLogBasedStoreResult(blockId, numRecords, walRecordHandle)
          }

## Kafka Direct API

    val kafkaParams = Map[String, String](
      "bootstrap.servers" -> "localhost:9092,anotherhost:9092",
      "group.id" -> "use_a_separate_group_id_for_each_stream",
      "auto.offset.reset" -> "latest"
    )
    val topics = Set("topicA", "topicB")
    val stream = KafkaUtils.createDirectStream(ssc,kafkaParams,topics)

1. createDirectStream 创建的 DirectKafkaInputDStream 对象继承自 InputDStream 而非是 ReceiverInputDStream，这一点很重要因为这种 Direct 方式并不会使用 receiver 接收数据，也就没有必要使用 receiverTracker 去启动 receiver 了。而代码中判断是不是需要启动 Receiver 的依据正是是否继承自 ReceiverInputDStream

          def getReceiverInputStreams(): Array[ReceiverInputDStream[_]] = this.synchronized {
            inputStreams.filter(_.isInstanceOf[ReceiverInputDStream[_]])
              .map(_.asInstanceOf[ReceiverInputDStream[_]])
              .toArray
          }

2. 在使用 DStream 生成 RDD 时最终会调用到 DirectKafkaInputDStream 的 compute 方法，而在这个方法中创建的是一个 KafkaRDD，然后就到了关键的地方了, 考虑 RDD 的调用链，每个 RDD 都需要前一个 RDD 返回的 Iterator. 看下面的方法：

          override def compute(thePart: Partition, context: TaskContext): Iterator[R] = {
            val part = thePart.asInstanceOf[KafkaRDDPartition]
            assert(part.fromOffset <= part.untilOffset, errBeginAfterEnd(part))
            if (part.fromOffset == part.untilOffset) {
              log.info(s"Beginning offset ${part.fromOffset} is the same as ending offset " +
                s"skipping ${part.topic} ${part.partition}")
              Iterator.empty
            } else {
              new KafkaRDDIterator(part, context)
            }
          }

    在 KafkaRDDIterator 会创建针对 topic 的 consumer，然后就可以直接从 Kafka 中取数据了，这里相当于把 Kafka 当成类似于 HDFS 一样的数据源。

## 忽略

最后，如果应用的实时性需求大于准确性，那么一块数据丢失后我们也可以选择忽略、不恢复失效的源头数据。

忽略可以分为细粒度忽略和粗粒度忽略，具体请看
[https://github.com/lw-lin/CoolplaySpark/blob/master/Spark%20Streaming%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%B3%BB%E5%88%97/4.1%20Executor%20%E7%AB%AF%E9%95%BF%E6%97%B6%E5%AE%B9%E9%94%99%E8%AF%A6%E8%A7%A3.md#4-忽略](https://github.com/lw-lin/CoolplaySpark/blob/master/Spark%20Streaming%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%B3%BB%E5%88%97/4.1%20Executor%20%E7%AB%AF%E9%95%BF%E6%97%B6%E5%AE%B9%E9%94%99%E8%AF%A6%E8%A7%A3.md#4-忽略)

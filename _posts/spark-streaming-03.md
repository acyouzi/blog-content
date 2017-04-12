---
title: Spark源码-streaming-03-driver 可靠性
date: 2017-04-12 16:21:12
tags:
    - spark
---
## 总结
1. ReceivedBlockTracker 管理 block 数据，如果配置了 checkpoint 目录，会 ReceivedBlockTracker 会把它管理的数据使用预写日志写入到 HDFS 中
2. driver 的状态也会在特定的阶段 Checkpoint 到 checkpoint 目录中去
3. 在编写 spark streaming 程序时，创建 StreamingContext 可以预先判断有没有 checkpoint 数据，这样在 driver 因故障重启时可以读取 checkpoint 数据恢复到重启前的状态

## ReceivedBlockTracker
ReceiverTracker 在实例化时会创建一个 receivedBlockTracker，这个对象负责实际的 block 信息管理。

在这个类的构造函数中可能会创建一个 WriteAheadLog 实例：

      private val writeAheadLogOption = createWriteAheadLog()
      
      private def createWriteAheadLog(): Option[WriteAheadLog] = {
        checkpointDirOption.map { checkpointDir =>
          val logDir = ReceivedBlockTracker.checkpointDirToLogDir(checkpointDirOption.get)
          WriteAheadLogUtils.createLogForDriver(conf, logDir, hadoopConf)
        }
      }

然后下面看一个具体的方法：

      def addBlock(receivedBlockInfo: ReceivedBlockInfo): Boolean = {
        try {
          // 如果配置了 wal 日志，先写到 wal 日志中
          // wal 日志持久化到 hdfs 中
          val writeResult = writeToLog(BlockAdditionEvent(receivedBlockInfo))
          if (writeResult) {
            synchronized {
              getReceivedBlockQueue(receivedBlockInfo.streamId) += receivedBlockInfo
            }
            logDebug(s"Stream ${receivedBlockInfo.streamId} received " +
              s"block ${receivedBlockInfo.blockStoreResult.blockId}")
          } else {
            logDebug(s"Failed to acknowledge stream ${receivedBlockInfo.streamId} receiving " +
              s"block ${receivedBlockInfo.blockStoreResult.blockId} in the Write Ahead Log.")
          }
          writeResult
        } catch {
          case NonFatal(e) =>
            logError(s"Error adding block $receivedBlockInfo", e)
            false
        }
      }

## Checkpoint
1. 每次 JobGenerator#generateJobs 完成 job 提交时会发送 DoCheckpoint 消息触发 Checkpoint

          private def generateJobs(time: Time) {
            // Checkpoint all RDDs marked for checkpointing to ensure their lineages are
            // truncated periodically. Otherwise, we may run into stack overflows (SPARK-6847).
            ssc.sparkContext.setLocalProperty(RDD.CHECKPOINT_ALL_MARKED_ANCESTORS, "true")
            Try {
              jobScheduler.receiverTracker.allocateBlocksToBatch(time) // allocate received blocks to batch
              graph.generateJobs(time) // generate jobs using allocated block
            } match {
              case Success(jobs) =>
                val streamIdToInputInfos = jobScheduler.inputInfoTracker.getInfo(time)
                jobScheduler.submitJobSet(JobSet(time, jobs, streamIdToInputInfos))
              case Failure(e) =>
                jobScheduler.reportError("Error generating jobs for time " + time, e)
                PythonDStream.stopStreamingContextIfPythonProcessIsDead(e)
            }
            eventLoop.post(DoCheckpoint(time, clearCheckpointDataLater = false))
          }

2. jobScheduler.submitJobSet 时，分别提交每个 job, 在每个 job 完成时会发送 JobCompleted 事件，这个事件最终也会触发 DoCheckpoint 事件

          private def clearMetadata(time: Time) {
            ssc.graph.clearMetadata(time)
        
            // If checkpointing is enabled, then checkpoint,
            // else mark batch to be fully processed
            if (shouldCheckpoint) {
              eventLoop.post(DoCheckpoint(time, clearCheckpointDataLater = true))
            } else {
              // If checkpointing is not enabled, then delete metadata information about
              // received blocks (block data not saved in any case). Otherwise, wait for
              // checkpointing of this batch to complete.
              val maxRememberDuration = graph.getMaxInputStreamRememberDuration()
              jobScheduler.receiverTracker.cleanupOldBlocksAndBatches(time - maxRememberDuration)
              jobScheduler.inputInfoTracker.cleanup(time - maxRememberDuration)
              markBatchFullyProcessed(time)
            }
          }

3. doCheckpoint 方法把一个 Checkpoint 对象持久化到 HDFS,同时如果有配置还会清理早期数据

          private def doCheckpoint(time: Time, clearCheckpointDataLater: Boolean) {
            if (shouldCheckpoint && (time - graph.zeroTime).isMultipleOf(ssc.checkpointDuration)) {
              logInfo("Checkpointing graph for time " + time)
              ssc.graph.updateCheckpointData(time)
              checkpointWriter.write(new Checkpoint(ssc, time), clearCheckpointDataLater)
            } else if (clearCheckpointDataLater) {
              markBatchFullyProcessed(time)
            }
          }
    
    Checkpoint 缓存了如下一些数据：
    
          val master = ssc.sc.master
          val framework = ssc.sc.appName
          val jars = ssc.sc.jars
          val graph = ssc.graph
          val checkpointDir = ssc.checkpointDir
          val checkpointDuration = ssc.checkpointDuration
          val pendingTimes = ssc.scheduler.getPendingTimes().toArray
          val sparkConfPairs = ssc.conf.getAll
          

## driver 重启

    // Function to create and setup a new StreamingContext
    def functionToCreateContext(): StreamingContext = {
      val ssc = new StreamingContext(...)   // new context
      val lines = ssc.socketTextStream(...) // create DStreams
      ...
      ssc.checkpoint(checkpointDirectory)   // set checkpoint directory
      ssc
    }
     
    // Get StreamingContext from checkpoint data or create a new one
    val context = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext _)

如上配置可以使得在重启时优先读取 checkpoint 目录中的数据
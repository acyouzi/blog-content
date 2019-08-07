---
title: Spark源码-streaming-01-Job 生成
date: 2017-04-05 21:35:47
description: 
tags:
	- spark
---
## 总结
1. 在看代码之前一直以为 Dstream 继承自 RDD, 原来他们之间并没有继承关系，DStream 是在 jobGenerator 时才创建了计算所需的 RDD。
2. 发现一个 github 项目超级棒，讲 Streaming 源码的，看源码的时候可以对照那里面的文章，地址是：

    [https://github.com/lw-lin/CoolplaySpark](https://github.com/lw-lin/CoolplaySpark)

## 流程
1. JobGenerator 的构造方法中创建了一个 RecurringTimer 定时发送 GenerateJobs 消息
        
        private val timer = new RecurringTimer(clock, ssc.graph.batchDuration.milliseconds,
        longTime => eventLoop.post(GenerateJobs(new Time(longTime))), "JobGenerator")

    而在 JobGenerator#start 方法中又启动了一个线程来负责处理发送给 JobGenerator 的消息
    
        eventLoop = new EventLoop[JobGeneratorEvent]("JobGenerator") {
          override protected def onReceive(event: JobGeneratorEvent): Unit = processEvent(event)
    
          override protected def onError(e: Throwable): Unit = {
            jobScheduler.reportError("Error in job generator", e)
          }
        }
        eventLoop.start()

2. 下面直接看处理 JobGenerator 消息的具体方法：JobGenerator#generateJobs

          private def generateJobs(time: Time) {
            // Checkpoint all RDDs marked for checkpointing to ensure their lineages are
            // truncated periodically. Otherwise, we may run into stack overflows (SPARK-6847).
            ssc.sparkContext.setLocalProperty(RDD.CHECKPOINT_ALL_MARKED_ANCESTORS, "true")
            Try {
              // 使用底层的 receivedBlockTracker 分配 blocks
              jobScheduler.receiverTracker.allocateBlocksToBatch(time) 
              // 生成 job
              // 下面重点看这个方法
              graph.generateJobs(time)
            } match {
              case Success(jobs) =>
                val streamIdToInputInfos = jobScheduler.inputInfoTracker.getInfo(time)
                // 这个方法会在一个新的线程中调用 job 的 run 方法
                jobScheduler.submitJobSet(JobSet(time, jobs, streamIdToInputInfos))
              case Failure(e) =>
                jobScheduler.reportError("Error generating jobs for time " + time, e)
                PythonDStream.stopStreamingContextIfPythonProcessIsDead(e)
            }
            eventLoop.post(DoCheckpoint(time, clearCheckpointDataLater = false))
          }

3. DStreamGraph.generateJobs(time)

          def generateJobs(time: Time): Seq[Job] = {
            logDebug("Generating jobs for time " + time)
            val jobs = this.synchronized {
              // 可以看到有几个 outputStream 就有几个 job
              outputStreams.flatMap { outputStream =>
                // 注意这里调用的是 ForEachDStream.generateJob
                val jobOption = outputStream.generateJob(time)
                jobOption.foreach(_.setCallSite(outputStream.creationSite))
                jobOption
              }
            }
            logDebug("Generated " + jobs.length + " jobs for time " + time)
            jobs
          }
    
    ForEachDStream.generateJob 中如果这个 rdd 曾经生成过则复用，否则重新计算。
    
          override def generateJob(time: Time): Option[Job] = {
            // 下面看 getOrCompute 方法
            parent.getOrCompute(time) match {
              case Some(rdd) =>
                // 这个函数对应到 RDD 里面应该算是 ResultRDD 的处理函数
                // foreachFunc 是一些基于 ForEachDStream 的算子传入的函数
                val jobFunc = () => createRDDWithLocalProperties(time, displayInnerRDDOps) {
                  foreachFunc(rdd, time)
                }
                Some(new Job(time, jobFunc))
              case None => None
            }
          }

4. getOrCompute 方法

          private[streaming] final def getOrCompute(time: Time): Option[RDD[T]] = {
            // 如果 RDD 已经生成过了直接使用以前的
            // 否则重新计算
            // generatedRDDs 是一个 HashMap，以 time 为键
            generatedRDDs.get(time).orElse {
              // Compute the RDD if time is valid (e.g. correct time in a sliding window)
              // of RDD generation, else generate nothing.
              if (isTimeValid(time)) {
                val rddOption = createRDDWithLocalProperties(time, displayInnerRDDOps = false) {
                  // 不检查输出目录的合法性(目录是否存在？)
                  PairRDDFunctions.disableOutputSpecValidation.withValue(true) {
                    // 注意前边 ForEachDStream.generateJob 调用的是 parent.getOrCompute
                    // 所以这个 compute 应该是 ForEachDStream 上一个依赖的方法
                    // compute 有多种实现，后面会看几个
                    compute(time)
                  }
                }
                
                rddOption.foreach { case newRDD =>
                  // Register the generated RDD for caching and checkpointing
                  if (storageLevel != StorageLevel.NONE) {
                    newRDD.persist(storageLevel)
                    logDebug(s"Persisting RDD ${newRDD.id} for time $time to $storageLevel")
                  }
                  if (checkpointDuration != null && (time - zeroTime).isMultipleOf(checkpointDuration)) {
                    newRDD.checkpoint()
                    logInfo(s"Marking RDD ${newRDD.id} for time $time for checkpointing")
                  }
                  generatedRDDs.put(time, newRDD)
                }
                rddOption
              } else {
                None
              }
            }
          }

5. compute 方法

    - ReceiverInputDStream#compute 
    
          // 这个方法拿的是 reciver 接收的数据
          // 创建的是 blockRDD
          override def compute(validTime: Time): Option[RDD[T]] = {
            val blockRDD = {
        
              if (validTime < graph.startTime) {
                // If this is called for any time before the start time of the context,
                // then this returns an empty RDD. This may happen when recovering from a
                // driver failure without any write ahead log to recover pre-failure data.
                new BlockRDD[T](ssc.sc, Array.empty)
              } else {
                // Otherwise, ask the tracker for all the blocks that have been allocated to this stream
                // for this batch
                val receiverTracker = ssc.scheduler.receiverTracker
                val blockInfos = receiverTracker.getBlocksOfBatch(validTime).getOrElse(id, Seq.empty)
        
                // Register the input blocks information into InputInfoTracker
                val inputInfo = StreamInputInfo(id, blockInfos.flatMap(_.numRecords).sum)
                ssc.scheduler.inputInfoTracker.reportInfo(validTime, inputInfo)
        
                // 根据 wal 日志是否开启以及块是否为空分别处理创建 RDD
                createBlockRDD(validTime, blockInfos)
              }
            }
            Some(blockRDD)
          }
    
    - ShuffledDStream#compute
    
          override def compute(validTime: Time): Option[RDD[(K, C)]] = {
            // 一般 RDD 的这个方法都是下面的流程，拿到父类的数据，然后处理
            parent.getOrCompute(validTime) match {
              case Some(rdd) => Some(rdd.combineByKey[C](
                  createCombiner, mergeValue, mergeCombiner, partitioner, mapSideCombine))
              case None => None
            }
          }
    
6. 拿到 Job 之后就该运行 Job 了, 也就是这条语句 :  jobScheduler.submitJobSet(JobSet(time, jobs, streamIdToInputInfos))

          def submitJobSet(jobSet: JobSet) {
            if (jobSet.jobs.isEmpty) {
              logInfo("No jobs added for time " + jobSet.time)
            } else {
              listenerBus.post(StreamingListenerBatchSubmitted(jobSet.toBatchInfo))
              jobSets.put(jobSet.time, jobSet)
              // jobExecutor 是线程池，默认大小是 1 , 可以通过 spark.streaming.concurrentJobs 设置
              // JobHandler 的 run 方法做了诸多配置，最终调用了 Job 的 run 方法
              jobSet.jobs.foreach(job => jobExecutor.execute(new JobHandler(job)))
              logInfo("Added jobs for time " + jobSet.time)
            }
          }
    
    Job#run
    
          def run() {
            // func 就是创建 Job 时传入的 foreachFunc
            _result = Try(func())
          }

7. 那么下一问题是 job 在哪里提交的？DStream 有一些 output 算子, 这些算子会调用 RDD 的 action 算子，然后就能触发 job 提交了。比如 print 方法

          def print(num: Int): Unit = ssc.withScope {
            def foreachFunc: (RDD[T], Time) => Unit = {
              (rdd: RDD[T], time: Time) => {
                // action 算子
                val firstNum = rdd.take(num + 1)
                // scalastyle:off println
                println("-------------------------------------------")
                println(s"Time: $time")
                println("-------------------------------------------")
                firstNum.take(num).foreach(println)
                if (firstNum.length > num) println("...")
                println()
                // scalastyle:on println
              }
            }
            foreachRDD(context.sparkContext.clean(foreachFunc), displayInnerRDDOps = false)
          }
        
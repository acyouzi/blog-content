---
title: Spark源码-Core-05-persist&&checkpoint&&blockManager
date: 2017-03-26 10:40:48
description: 
tags:
	- spark
---
1. cache 方法实际上调用的是 MEMORY_ONLY level 的 Persist
2. 设置过 persist 的 rdd 在调用时会在实际计算时调用 getOrCompute,如果存在直接从缓存中拿结果，否则重新计算并交给 blockManager 缓存
3. checkpoint 使用之前需要先调用 sc.setCheckpointDir() 设置缓存目录.目录最好设置在分布式文件系统上
4. checkpoint 执行的时机是在任务 rdd 执行结束之后在 sc.runJob 中调用最后一个 rdd 的 doCheckpoint 方法
5. doCheckpoint 方法会重新提交 job 运行以存储 rdd,因为 rdd 的计算逻辑时如果存在 persist 数据直接从缓存中取，所以最好在想要 checkpoint 的地方先调用 persist, 否则会重新计算这个 rdd 的数据
6. checkpoint 会改变 rdd 的 lineage
7. Broadcast Variable 可以使得需要共享的变量在每个节点保存一份而不是在每个 task 上，减少了网络 io 和内存消耗。

## cache()

    Persist this RDD with the default storage level (`MEMORY_ONLY`)    

## persist()

      def persist(newLevel: StorageLevel): this.type = {
        if (isLocallyCheckpointed) {
          // 已经缓存过，transformStorageLevel 会给缓存级别上加一个磁盘
          persist(LocalRDDCheckpointData.transformStorageLevel(newLevel), allowOverride = true)
        } else {
          persist(newLevel, allowOverride = false)
        }
      }

在 iterator 会检查缓存级别，如果缓存了，先去缓存中拿。

      final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
        if (storageLevel != StorageLevel.NONE) {
          getOrCompute(split, context)
        } else {
          computeOrReadCheckpoint(split, context)
        }
      }

最终会调用 blockManager 的 getOrElseUpdate 方法

      def getOrElseUpdate[T](
          blockId: BlockId,
          level: StorageLevel,
          classTag: ClassTag[T],
          makeIterator: () => Iterator[T]): Either[BlockResult, Iterator[T]] = {
        // 从本地或者远程 blockmanager 读
        get[T](blockId)(classTag) match {
          case Some(block) =>
            return Left(block)
          case _ =>
            // Need to compute the block.
        }
        // 重新计算，重新缓存
        // makeIterator 是重新计算的函数
        // 形式 
        // () => {
        //   readCachedBlock = false
        //   // 重新计算
        //   computeOrReadCheckpoint(partition, context)
        // }
        // doPutIterator 略长，就不贴了主要做了如下事情
        //  1. 根据设置的内存磁盘缓存级别，还有是否序列化做处理
        //  2. 报告 blockManagerMaster 状态变化 
        doPutIterator(blockId, makeIterator, level, classTag, keepReadLock = true) match {
          case None =>
            // doPut() 啥都没做，可能多线程?别的线程做完了?
            val blockResult = getLocalValues(blockId).getOrElse {
              releaseLock(blockId)
              throw new SparkException(s"get() failed for block $blockId even though we held a lock")
            }
            // 释放锁，返回结果
            releaseLock(blockId)
            Left(blockResult)
          case Some(iter) =>
            // 数据太大内存放不下， dropped 到磁盘，通过迭代器返回
           Right(iter)
        }
      }

    下面是 get 方法
    
         def get[T: ClassTag](blockId: BlockId): Option[BlockResult] = {
            val local = getLocalValues(blockId)
            if (local.isDefined) {
              logInfo(s"Found block $blockId locally")
              return local
            }
            // 远程的要连接远程节点的 blockManager 获取
            // 要通过 blockManagerMaster 获取 block 在哪些节点上
            // 然后再从远程节点的 blockManager 上获取
            val remote = getRemoteValues[T](blockId)
            if (remote.isDefined) {
              logInfo(s"Found block $blockId remotely")
              return remote
            }
            None
          }

## checkpoint()
    
      def checkpoint(): Unit = RDDCheckpointData.synchronized {
        if (context.checkpointDir.isEmpty) {
          throw new SparkException("Checkpoint directory has not been set in the SparkContext")
        } else if (checkpointData.isEmpty) {
          // 初始化
          checkpointData = Some(new ReliableRDDCheckpointData(this))
        }
      }

然后在 sparkContext 的 runJob 方法，注意最后一行，在 job 执行完成之后调用 rdd 的 doCheckpoint 方法。
    
      def runJob[T, U: ClassTag](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      resultHandler: (Int, U) => Unit): Unit = {
        if (stopped.get()) {
          throw new IllegalStateException("SparkContext has been shutdown")
        }
        val callSite = getCallSite
        val cleanedFunc = clean(func)
        logInfo("Starting job: " + callSite.shortForm)
        if (conf.getBoolean("spark.logLineage", false)) {
          logInfo("RDD's recursive dependencies:\n" + rdd.toDebugString)
        }
        dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
        progressBar.foreach(_.finishAll())
        rdd.doCheckpoint()
      }

再看 doCheckpoint 方法

      private[spark] def doCheckpoint(): Unit = {
        RDDOperationScope.withScope(sc, "checkpoint", allowNesting = false, ignoreParent = true) {
          if (!doCheckpointCalled) {
            doCheckpointCalled = true
            // 递归的往前找，直到有一个定义了 checkpointData
            if (checkpointData.isDefined) {
              if (checkpointAllMarkedAncestors) {
                // 继续往下走
                // 从第一个 checkpoint 的地方开始调用 checkpointData.get.checkpoint()
                dependencies.foreach(_.rdd.doCheckpoint())
              }
              checkpointData.get.checkpoint()
            } else {
              dependencies.foreach(_.rdd.doCheckpoint())
            }
          }
        }
      }
    
下面是 RDDCheckpointData#checkpoint 方法

      final def checkpoint(): Unit = {
        // Guard against multiple threads checkpointing the same RDD by
        // atomically flipping the state of this RDDCheckpointData
        RDDCheckpointData.synchronized {
          if (cpState == Initialized) {
            // 标记状态为 CheckpointingInProgress
            cpState = CheckpointingInProgress
          } else {
            return
          }
        }
        
        // 新建一个 RDD
        val newRDD = doCheckpoint()
    
        // 更改状态为 Checkpointed，并且改变依赖关系(lineage)
        RDDCheckpointData.synchronized {
          cpRDD = Some(newRDD)
          cpState = Checkpointed
          // 清理 rdd 原有的依赖
          rdd.markCheckpointed()
        }
      }
    
这个是 doCheckpoint 方法创建 ReliableCheckpointRDD 的代码，在这里面完成了 checkpoint 文件存储操作。

      def writeRDDToCheckpointDirectory[T: ClassTag](
          originalRDD: RDD[T],
          checkpointDir: String,
          blockSize: Int = -1): ReliableCheckpointRDD[T] = {
        val sc = originalRDD.sparkContext
        
        // Create the output path for the checkpoint
        val checkpointDirPath = new Path(checkpointDir)
        val fs = checkpointDirPath.getFileSystem(sc.hadoopConfiguration)
        if (!fs.mkdirs(checkpointDirPath)) {
          throw new SparkException(s"Failed to create checkpoint path $checkpointDirPath")
        }
    
        // Save to file, and reload it as an RDD
        val broadcastedConf = sc.broadcast(
          new SerializableConfiguration(sc.hadoopConfiguration))
        
        // 划重点 -- writePartitionToCheckpointFile 就是把数据写的持久化目录中的方法
        // 比如写到 hdfs 的某个目录
        // 这个 runjob 是重新提交 job 
        // 这也是为什么想要 checkpoint 的节点最好先 persist 一下的原因。
        // 如果不 persist 那么需要 checkpoint 的数据需要从头跑一遍。
        // writePartitionToCheckpointFile 的方法就不贴了, 没什么意思
        sc.runJob(originalRDD,
          writePartitionToCheckpointFile[T](checkpointDirPath.toString, broadcastedConf) _)
        if (originalRDD.partitioner.nonEmpty) {
          writePartitionerToCheckpointDir(sc, originalRDD.partitioner.get, checkpointDirPath)
        }
        val newRDD = new ReliableCheckpointRDD[T](
          sc, checkpointDirPath.toString, originalRDD.partitioner)
        if (newRDD.partitions.length != originalRDD.partitions.length) {
          throw new SparkException(
            s"Checkpoint RDD $newRDD(${newRDD.partitions.length}) has different " +
              s"number of partitions from original RDD $originalRDD(${originalRDD.partitions.length})")
        }
        newRDD
      }
    
下面看一看 checkpoint rdd 是怎么起作用的：
    
      private[spark] def computeOrReadCheckpoint(split: Partition, context: TaskContext): Iterator[T] =
      {
        // isCheckpointedAndMaterialized 在checkpoint 过的节点返回为真
        if (isCheckpointedAndMaterialized) {
          // 比较关键的是 firstParent 怎么的到的，具体看下面的方法
          firstParent[T].iterator(split, context)
        } else {
          compute(split, context)
        }
      }
      
在 firstParent 方法中会去拿依赖的 head, 依赖获取的方法如下： 

      final def dependencies: Seq[Dependency[_]] = {
        // 如果 checkpoint 有值直接返回
        checkpointRDD.map(r => List(new OneToOneDependency(r))).getOrElse {
          if (dependencies_ == null) {
            dependencies_ = getDependencies
          }
          dependencies_
        }
      }

到这里 checkpoint 的原理机制就介绍完了

## 闭包处理
如果看过 spark 算子实现的代码你肯定会疑惑在每个 spark 算子中都会出现的下面这句话到底做了什么 

    val cleanF = sc.clean(f)

这里辗转调用最终调用到一个非常长的方法 ClosureCleaner#clean, clean 方法内部使用了 asm 代码库直接分析字节码去确定闭包引用,然后在每个 task 上都会 copy 一份闭包引用，也就是说每个 task 有一份变量实例。因为 asm 库不太熟悉，等以后再补上代码分析..

## Broadcast Variable && Accumulator
就不看代码了，看吐了，简单介绍下特点

一个算子的函数中使用到了某个外部的变量，那么这个变量的值会被拷贝到每个task中，此时每个task只能操作自己的那份变量副本，也就是闭包清理步骤做的事情。如果想要在多个 task 间共享一份数据这种方法是不恰当的，应该使用 Broadcast Variable（广播变量）或者 Accumulator（累加变量），Broadcast Variable会将使用到的变量仅在每个节点拷贝一份，而不是在每个 task 上(底层使用 blockManager 实现)，减少网络传输以及内存消耗。

Accumulator 提供了累加的功能，但是task只能对Accumulator进行累加操作，不能读取它的值，只有Driver程序可以读取Accumulator的值。

## blockManager 
这是 spark 中非常底层的部分，shuffle、persist、Broadcast 等都要用到它，blockMangerMaster 在 Driver 上启动，负责保存各个节点上 block 的状态每个 executor 上有 blockManager 负责管理本节点上的数据缓存，以及状态到 BlockManagerMaster 的报备。

... 有空再补上吧, 找实习烦死了
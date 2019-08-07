---
title: Spark源码-Core-04-shuffle过程
date: 2017-03-23 01:11:35
description: 
tags:
	- spark
---
## 总结
1. shuffle 过程分为 shuffle write 和 shuffle read 两个阶段
2. 除了 stage0 外，每个stage 都是以 ShuffledRDD 开头的，通过 ShuffledRDD 完成 shuffle read 操作
3. Task 分为两类(ShuffleMapTask, ResultTask), 重点需要关注的是 ShuffleMapTask
4. shuffle write 中首先需要明确的是 task 中引用的 rdd 是 ShuffledRDD 的前一个 rdd. ShuffleMapTask 把前一个 rdd 的输出写入到磁盘完成 shuffle write 操作
5. 在 spark 2.1.0 中 应该是只有 SortShuffleManager 一种 manager ( 网上介绍早些版本还有一个 HashShuffleManage)
6. ShuffleManager 中有 SortShuffleWriter 和 BypassMergeSortShuffleWriter, 区别主要在于有没有内存排序
7. spark.shuffle.sort.bypassMergeThreshold (defalut:200) 参数控制使用 SortShuffleWriter 或者 BypassMergeSortShuffleWriter
8. SortShuffleManager 的磁盘生成文件最终只有两个，一个是合并后的数据文件，一个是索引文件
9. SortShuffleWriter 在写文件时会先在内存中缓存，当达到一定值后才会溢写到文件
10. SortShuffleWriter 最后的 spillFile 合并过程多看几遍，挺有意思的。
11. spark.shuffle.file.buffer 控制 shuffle write 的内存缓存大小
12. spark.reducer.maxSizeInFlight 参数控制 shuffle read 每次拉取数据的大小，默认 48m
13. shuffle read 根据是否需要聚合，是否需要数据有序在数据拉取过来之后会发生两次 Combine

## 流程

1. task 提交到 executor 上实际上是被 org.apache.spark.executor.CoarseGrainedExecutorBackend#receive 处理，经过多层辗转调用最后调用到 org.apache.spark.executor.Executor.TaskRunner#run ,在此方法中完成了 Task 实例化, Task 调用, 结果返回, 资源释放等操作. 当然这都不是重点，重点是他调用了 Task 的 run 方法，而 Task 就目前来看已经见过 ShuffleMapTask 和 ResultTask 两类了, 在此方法中调用的 runTask 方法 ShuffleMapTask 和 ResultTask 分别有不同的实现。下面先来看 ShuffleMapTask#runTask

          override def runTask(context: TaskContext): MapStatus = {
            // 此处省略的一些初始化代码
            ...
            
            // shuffleManager 
            var writer: ShuffleWriter[Any, Any] = null
            try {
              val manager = SparkEnv.get.shuffleManager
              // dep.shuffleHandle 根据设置返回 shuffer 类型
              // 下面首先看 dep.shuffleHandle 然后在看 manager.getWriter
              writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
              // 理解 shuffle 很重要的理解这个 rdd 到底是哪个 rdd
              // 下面会写一点关于这个 rdd
              writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
              writer.stop(success = true).get
            } catch {
              case e: Exception =>
                try {
                  if (writer != null) {
                    writer.stop(success = false)
                  }
                } catch {
                  case e: Exception =>
                    log.debug("Could not stop writer", e)
                }
                throw e
            }
          }
    
    在 dep.shuffleHandle 最后调用 org.apache.spark.shuffle.sort.SortShuffleManager#registerShuffle 方法

          override def registerShuffle[K, V, C](
          shuffleId: Int,
          numMaps: Int,
          dependency: ShuffleDependency[K, V, C]): ShuffleHandle = {
            if (SortShuffleWriter.shouldBypassMergeSort(SparkEnv.get.conf, dependency)) {
              // 在 dep.partitioner.numPartitions 小于等于 bypassMergeThreshold 时返回 true 返回 BypassMergeSortShuffleHandle
              // 在 mapSideCombine 为 true 时不使用 BypassMergeSortShuffleHandle
              new BypassMergeSortShuffleHandle[K, V](
                shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
            } else if (SortShuffleManager.canUseSerializedShuffle(dependency)) {
              // 序列化类型
              new SerializedShuffleHandle[K, V](
                shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
            } else {
              new BaseShuffleHandle(shuffleId, numMaps, dependency)
            }
          }
    
    下面来看 org.apache.spark.shuffle.sort.SortShuffleManager#getWriter 方法

          override def getWriter[K, V](
          handle: ShuffleHandle,
          mapId: Int,
          context: TaskContext): ShuffleWriter[K, V] = {
            numMapsForShuffle.putIfAbsent(
              handle.shuffleId, handle.asInstanceOf[BaseShuffleHandle[_, _, _]].numMaps)
            val env = SparkEnv.get
            // 根据 handle 的类型返回 ShuffleWriter
            handle match {
              case unsafeShuffleHandle: SerializedShuffleHandle[K @unchecked, V @unchecked] =>
                new UnsafeShuffleWriter(
                  env.blockManager,
                  shuffleBlockResolver.asInstanceOf[IndexShuffleBlockResolver],
                  context.taskMemoryManager(),
                  unsafeShuffleHandle,
                  mapId,
                  context,
                  env.conf)
              case bypassMergeSortHandle: BypassMergeSortShuffleHandle[K @unchecked, V @unchecked] =>
                new BypassMergeSortShuffleWriter(
                  env.blockManager,
                  shuffleBlockResolver.asInstanceOf[IndexShuffleBlockResolver],
                  bypassMergeSortHandle,
                  mapId,
                  context,
                  env.conf)
              case other: BaseShuffleHandle[K @unchecked, V @unchecked, _] =>
                new SortShuffleWriter(shuffleBlockResolver, other, mapId, context)
            }
          }
    
2. 这里先简单介绍一下 SortShuffleWriter 和 BypassMergeSortShuffleWriter 两种 Writer. 
    
    SortShuffleWriter 会在内存中排序，然后可能会产生多个溢写文件,在最后合并为一个文件。

    BypassMergeSortShuffleWriter 省去了内存中排序的过程，每个 task 创建一个临时文件，直接把数据写入临时文件，在最后阶段也有合并操作。   

3. shuffle 第三步要做的事情是调用 rdd.iterator 处理数据，然后通过 writer 把数据写到磁盘。这里很重要的一点是搞明白这个 rdd 到底是哪个类的实例，是 shufflerdd 还是 mappartitionrdd? 这个问题的重点还是在 stage 的划分上，下面从新看 stage 的划分.

    在 DAGScheduler#createShuffleMapStage 方法中创建 ShuffleMapStage 的时候传入的 rdd 引用是通过 shuffleDep 参数传入的，而这个参数是就是 RDD.dependencies 中类型为 ShuffleDependency 的参数。
    
    那再看一看 ShuffleDependency 是怎么创建的，在 org.apache.spark.rdd.ShuffledRDD 类中的如下方法。

          override def getDependencies: Seq[Dependency[_]] = {
            val serializer = userSpecifiedSerializer.getOrElse {
              val serializerManager = SparkEnv.get.serializerManager
              if (mapSideCombine) {
                serializerManager.getSerializer(implicitly[ClassTag[K]], implicitly[ClassTag[C]])
              } else {
                serializerManager.getSerializer(implicitly[ClassTag[K]], implicitly[ClassTag[V]])
              }
            }
            // 这里 ShuffleDependency 创建的时候传入的 prev 是什么呢
            // prev 是在 rdd 创建的时候传入的上一个 rdd 的引用
            // 比如 map(xxx).reduceByKey(xxx)
            // map方法创建的 MapPartitionsRDD 调用 reduceByKey 
            // reduceByKey 创建 ShuffledRDD 会在 prev 位置传入 this 指针
            List(new ShuffleDependency(prev, part, serializer, keyOrdering, aggregator, mapSideCombine))
          }
    
    到这里也就能看到 stage 划分中 shuffle Writer 过程结束在 shuffledRdd 的前一个 RDD, 而 shuffledRdd 则是作为下一个 stage shuffled Read 的开头。

4. 搞清楚了 shuffle writer 调用的是哪个 rdd 下面就要看看 rdd 之间是怎么串联起来的了

          final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
            // 如果有保存运行结果先尝试获取缓存
            if (storageLevel != StorageLevel.NONE) {
              getOrCompute(split, context)
            } else {
              computeOrReadCheckpoint(split, context)
            }
          }
    
          private[spark] def computeOrReadCheckpoint(split: Partition, context: TaskContext): Iterator[T] =
          {
            // 暂时不太懂这个 isCheckpointedAndMaterialized 是在做什么
            // 后面再看
            if (isCheckpointedAndMaterialized) {
              firstParent[T].iterator(split, context)
            } else {
              // rdd 的 compute
              compute(split, context)
            }
          }
    
    下面来看一下 MapPartitionsRDD#compute，这是众多 rdd 中的一个。
    
        override def compute(split: Partition, context: TaskContext): Iterator[U] =
        f(context, split.index, firstParent[T].iterator(split, context))
    
    这里会调用父依赖的 iterator 把输出传入到本 rdd 的处理函数，这样在一个 stage 中的 rdd 就串联起来了，每个 stage 开头为 shuffledRdd 或者是 stage0 开始阶段获取数据的第一个 rdd.

5. rdd 的输出结果会交给 write 方法，下面是 SortShuffleWriter 的 write

          /** Write a bunch of records to this task's output */
          override def write(records: Iterator[Product2[K, V]]): Unit = {
            // 创建 ExternalSorter
            // 根据创建 shuffleRdd 的 操作不同会设置为 true 或者 false
            sorter = if (dep.mapSideCombine) {
              require(dep.aggregator.isDefined, "Map-side combine without Aggregator specified!")
              new ExternalSorter[K, V, C](
                context, dep.aggregator, Some(dep.partitioner), dep.keyOrdering, dep.serializer)
            } else {
              new ExternalSorter[K, V, V](
                context, aggregator = None, Some(dep.partitioner), ordering = None, dep.serializer)
            }
            // 这个操作会缓存或把文件写的磁盘的临时文件中
            sorter.insertAll(records)
        
            // Don't bother including the time to open the merged output file in the shuffle write time,
            // because it just opens a single file, so is typically too fast to measure accurately
            // (see SPARK-3570).
            val output = shuffleBlockResolver.getDataFile(dep.shuffleId, mapId)
            val tmp = Utils.tempFileWith(output)
            try {
              val blockId = ShuffleBlockId(dep.shuffleId, mapId, IndexShuffleBlockResolver.NOOP_REDUCE_ID)
              // 合并为一个文件，返回文件合并后的索引
              val partitionLengths = sorter.writePartitionedFile(blockId, tmp)
              // 把索引写到索引文件中，也就是说 SortShuffleManager 最后只生成两个文件
              shuffleBlockResolver.writeIndexFileAndCommit(dep.shuffleId, mapId, partitionLengths, tmp)
              mapStatus = MapStatus(blockManager.shuffleServerId, partitionLengths)
            } finally {
              if (tmp.exists() && !tmp.delete()) {
                logError(s"Error while deleting temp file ${tmp.getAbsolutePath}")
              }
            }
          }
    
    这里面比较重要的是 insertAll 方法，在 insertAll 方法中会先缓存键值对，然后当超过一定数量会写入到磁盘的临时文件中。

          def insertAll(records: Iterator[Product2[K, V]]): Unit = {
            // 需不需要在shuffle write 端合并。
            val shouldCombine = aggregator.isDefined
            // 如果不需要合并会直接存到缓存里面
            // 如果需要合并会调用 changeValue 通过 update 函数更新
            if (shouldCombine) {
              // Combine values in-memory first using our AppendOnlyMap
              val mergeValue = aggregator.get.mergeValue
              val createCombiner = aggregator.get.createCombiner
              var kv: Product2[K, V] = null
              val update = (hadValue: Boolean, oldValue: C) => {
                if (hadValue) mergeValue(oldValue, kv._2) else createCombiner(kv._2)
              }
              while (records.hasNext) {
                addElementsRead()
                kv = records.next()
                // 向一个 AppendOnlyMap 中添加数据
                // 同时会采样集合大小，代码比较简单就不贴了
                map.changeValue((getPartition(kv._1), kv._1), update)
                maybeSpillCollection(usingMap = true)
              }
            } else {
              // Stick values into our buffer
              while (records.hasNext) {
                addElementsRead()
                val kv = records.next()
                // 如果不需要排序直接写到 buffer 里面
                buffer.insert(getPartition(kv._1), kv._1, kv._2.asInstanceOf[C])
                // 检查是不是需要写到磁盘
                maybeSpillCollection(usingMap = false)
              }
            }
          }
    
    下面再来看 maybeSpillCollection 方法：

          private def maybeSpillCollection(usingMap: Boolean): Unit = {
            var estimatedSize = 0L
            if (usingMap) {
              estimatedSize = map.estimateSize()
              if (maybeSpill(map, estimatedSize)) {
                map = new PartitionedAppendOnlyMap[K, C]
              }
            } else {
              estimatedSize = buffer.estimateSize()
              if (maybeSpill(buffer, estimatedSize)) {
                buffer = new PartitionedPairBuffer[K, C]
              }
            }
        
            if (estimatedSize > _peakMemoryUsedBytes) {
              _peakMemoryUsedBytes = estimatedSize
            }
          }
    
    上面的方法很简单，核心业务都放到 maybeSpill 中了
    
          protected def maybeSpill(collection: C, currentMemory: Long): Boolean = {
            var shouldSpill = false
            // 32 的倍数，并且大于等于 myMemoryThreshold
            // myMemoryThreshold 通过 "spark.shuffle.spill.initialMemoryThreshold" 配置
            // 默认值 5 * 1024 * 1024
            if (elementsRead % 32 == 0 && currentMemory >= myMemoryThreshold) {
              // 内存分配这块不太懂他是怎么处理的
              // 后面再说
              val amountToRequest = 2 * currentMemory - myMemoryThreshold
              val granted = acquireMemory(amountToRequest)
              myMemoryThreshold += granted
              // If we were granted too little memory to grow further (either tryToAcquire returned 0,
              // or we already had more memory than myMemoryThreshold), spill the current collection
              shouldSpill = currentMemory >= myMemoryThreshold
            }
            shouldSpill = shouldSpill || _elementsRead > numElementsForceSpillThreshold
            // 如果需要溢写到磁盘
            if (shouldSpill) {
              _spillCount += 1
              logSpillage(currentMemory)
              // 内存中的数据写入到本地磁盘文件
              // 并且写入的文件信息保存到 spills 变量中
              spill(collection)
              _elementsRead = 0
              _memoryBytesSpilled += currentMemory
              releaseMemory()
            }
            shouldSpill
          }
    
    在完成了 sorter.insertAll 之后就该合并文件了, 把前面产生的小文件合并到一个文件中。
    
          def writePartitionedFile(
          blockId: BlockId,
          outputFile: File): Array[Long] = {
    
            // Track location of each range in the output file
            val lengths = new Array[Long](numPartitions)
            val writer = blockManager.getDiskWriter(blockId, outputFile, serInstance, fileBufferSize,
              context.taskMetrics().shuffleWriteMetrics)
            if (spills.isEmpty) {
              // 没有磁盘溢写文件，则只把内存中的文件写入磁盘
              val collection = if (aggregator.isDefined) map else buffer
              val it = collection.destructiveSortedWritablePartitionedIterator(comparator)
              while (it.hasNext) {
                val partitionId = it.nextPartition()
                while (it.hasNext && it.nextPartition() == partitionId) {
                  it.writeNext(writer)
                }
                val segment = writer.commitAndGet()
                lengths(partitionId) = segment.length
              }
            } else {
              // 使用分区迭代器，合并多个磁盘溢写文件
              // 这个 partitionedIterator 怎么来的是一个比较重要的点
              for ((id, elements) <- this.partitionedIterator) {
                if (elements.hasNext) {
                  for (elem <- elements) {
                    writer.write(elem._1, elem._2)
                  }
                  val segment = writer.commitAndGet()
                  lengths(id) = segment.length
                }
              }
            }
            writer.close()
            context.taskMetrics().incMemoryBytesSpilled(memoryBytesSpilled)
            context.taskMetrics().incDiskBytesSpilled(diskBytesSpilled)
            context.taskMetrics().incPeakExecutionMemory(peakMemoryUsedBytes)
            lengths
          }
    
    下面看 partitionedIterator 的代码
    
          def partitionedIterator: Iterator[(Int, Iterator[Product2[K, C]])] = {
            val usingMap = aggregator.isDefined
            val collection: WritablePartitionedPairCollection[K, C] = if (usingMap) map else buffer
            if (spills.isEmpty) {
              // 如果只有内存中的数据没有磁盘溢写文件
              // 就是下边简单的分组
              if (!ordering.isDefined) {
                groupByPartition(destructiveIterator(collection.partitionedDestructiveSortedIterator(None)))
              } else {
                groupByPartition(destructiveIterator(
                  collection.partitionedDestructiveSortedIterator(Some(keyComparator))))
              }
            } else {
              // 把磁盘中的文件和内存中的文件做一个合并
              merge(spills, destructiveIterator(
                collection.partitionedDestructiveSortedIterator(comparator)))
            }
          }
    
    然后看 merge 的过程
    
          private def merge(spills: Seq[SpilledFile], inMemory: Iterator[((Int, K), C)])
          : Iterator[(Int, Iterator[Product2[K, C]])] = {
            // 得到所有磁盘文件的对应对象
            val readers = spills.map(new SpillReader(_))
            val inMemBuffered = inMemory.buffered
            (0 until numPartitions).iterator.map { p =>
              val inMemIterator = new IteratorForPartition(p, inMemBuffered)
              // 磁盘与内存数据到这里就等同对待了
              val iterators = readers.map(_.readNextPartition()) ++ Seq(inMemIterator)
              if (aggregator.isDefined) {
                // 是否执行聚合操作
                // mergeWithAggregation 实际上是对 mergeSort 的输出做了一层封装
                (p, mergeWithAggregation(
                  iterators, aggregator.get.mergeCombiners, keyComparator, ordering.isDefined))
              } else if (ordering.isDefined) {
                // sort the elements without trying to merge them
                (p, mergeSort(iterators, ordering.get))
              } else {
                (p, iterators.iterator.flatten)
              }
            }
          }
    
    下面再来看看 mergeSort 方法，这个方法挺有意思。
    
          private def mergeSort(iterators: Seq[Iterator[Product2[K, C]]], comparator: Comparator[K])
          : Iterator[Product2[K, C]] =
          {
            val bufferedIters = iterators.filter(_.hasNext).map(_.buffered)
            type Iter = BufferedIterator[Product2[K, C]]
            // bufferedIters 方法优先级队列中
            val heap = new mutable.PriorityQueue[Iter]()(new Ordering[Iter] {
              // 按照 bufferedIters head 的 key 比较
              override def compare(x: Iter, y: Iter): Int = -comparator.compare(x.head._1, y.head._1)
            })
            heap.enqueue(bufferedIters: _*)  // Will contain only the iterators with hasNext = true
            new Iterator[Product2[K, C]] {
              override def hasNext: Boolean = !heap.isEmpty
        
              override def next(): Product2[K, C] = {
                // 拿出 bufferedIters 中 head 元素的 key 最小的 bufferedIters 的 (k/v)
                // 因为前面 spill 本来就是按照 key 排序过的，所以这里每个 bufferedIters 内部都是有序的
                // leetcode 上貌似有个类似的编程题, easy 级别的。 
                if (!hasNext) {
                  throw new NoSuchElementException
                }
                val firstBuf = heap.dequeue()
                val firstPair = firstBuf.next()
                if (firstBuf.hasNext) {
                  heap.enqueue(firstBuf)
                }
                firstPair
              }
            }
          }
    
    最后再看一下 mergeWithAggregation 整个 shuffle writer 操作就完成了。因为代码略长，就只贴 Iterator 创建部分代码吧
          
          // 这部分代码位于不需要保证全局有序的分支中
          new Iterator[Iterator[Product2[K, C]]] {
            // 用到了 mergeSort 实际上就是对 mergeSort 的结果做了一层封装
            // 重点还是在 next 方法中
            val sorted = mergeSort(iterators, comparator).buffered
    
            // Buffers reused across elements to decrease memory allocation
            val keys = new ArrayBuffer[K]
            val combiners = new ArrayBuffer[C]
    
            override def hasNext: Boolean = sorted.hasNext
    
            override def next(): Iterator[Product2[K, C]] = {
              if (!hasNext) {
                throw new NoSuchElementException
              }
              keys.clear()
              combiners.clear()
              val firstPair = sorted.next()
              keys += firstPair._1
              combiners += firstPair._2
              val key = firstPair._1
              // 当  comparator.compare(sorted.head._1, key) == 0 不成立时结束循环
              // 也就是各个 spillFile 中没有与 key 等同的键了
              while (sorted.hasNext && comparator.compare(sorted.head._1, key) == 0) {
                val pair = sorted.next()
                var i = 0
                var foundKey = false
                // 循环查找，如果 key 在 kays 里面就聚合 val
                // 那什么情况会不在 keys 里面呢？
                // 上边的 comparator.compare 比较的是 hashcode
                // 这里是不是判断的 hashcode 相等但是 key 值不相等的情况呢？
                // 猜测...
                // 可是又为什么要比较 hashcode 呢？
                // 难道是出于效率考虑？
                // 奇怪啊，为啥不直接用 key 做比较
                while (i < keys.size && !foundKey) {
                  if (keys(i) == pair._1) {
                    combiners(i) = mergeCombiners(combiners(i), pair._2)
                    foundKey = true
                  }
                  i += 1
                }
                if (!foundKey) {
                  keys += pair._1
                  combiners += pair._2
                }
              }
    
              // 拉链操作键值合并
              keys.iterator.zip(combiners.iterator)
            }
          }.flatMap(i => i)
    
6. 因为前面已经知道 shuffle read 发生在 ShuffledRDD 中，所以直接看 ShuffledRDD 的 compute 方法。

          override def compute(split: Partition, context: TaskContext): Iterator[(K, C)] = {
            val dep = dependencies.head.asInstanceOf[ShuffleDependency[K, V, C]]
            // getReader 返回了一个 BlockStoreShuffleReader 对象
            // dep.shuffleHandle 与前面分析的一样
            SparkEnv.get.shuffleManager.getReader(dep.shuffleHandle, split.index, split.index + 1, context)
              .read()
              .asInstanceOf[Iterator[(K, C)]]
          }
    
    下一步调用 BlockStoreShuffleReader 的 read 方法，这是一个超长的方法
    
          override def read(): Iterator[Product2[K, C]] = {
            // ShuffleBlockFetcherIterator 负责从远端拿数据
            // 后面会具体分析
            val blockFetcherItr = new ShuffleBlockFetcherIterator(
              context,
              blockManager.shuffleClient,
              blockManager,
              mapOutputTracker.getMapSizesByExecutorId(handle.shuffleId, startPartition, endPartition),
              // Note: we use getSizeAsMb when no suffix is provided for backwards compatibility
              SparkEnv.get.conf.getSizeAsMb("spark.reducer.maxSizeInFlight", "48m") * 1024 * 1024,
              SparkEnv.get.conf.getInt("spark.reducer.maxReqsInFlight", Int.MaxValue))
        
            // Wrap the streams for compression and encryption based on configuration
            val wrappedStreams = blockFetcherItr.map { case (blockId, inputStream) =>
              serializerManager.wrapStream(blockId, inputStream)
            }
        
            val serializerInstance = dep.serializer.newInstance()
        
            // Create a key/value iterator for each stream
            val recordIter = wrappedStreams.flatMap { wrappedStream =>
              // 反序列化，组装成 key/val
              serializerInstance.deserializeStream(wrappedStream).asKeyValueIterator
            }
        
            // Update the context task metrics for each record read.
            // 又一层封装，scala 的封装还真是比 Java 简洁
            val readMetrics = context.taskMetrics.createTempShuffleReadMetrics()
            val metricIter = CompletionIterator[(Any, Any), Iterator[(Any, Any)]](
              recordIter.map { record =>
                readMetrics.incRecordsRead(1)
                record
              },
              context.taskMetrics().mergeShuffleReadMetrics())
        
            // An interruptible iterator must be used here in order to support task cancellation
            val interruptibleIter = new InterruptibleIterator[(Any, Any)](context, metricIter)
        
            val aggregatedIter: Iterator[Product2[K, C]] = if (dep.aggregator.isDefined) {
              // 是否已经在 map 端聚合过了
              // combineCombinersByKey、combineValuesByKey 底层调用的是 ExternalAppendOnlyMap
              // 是一个 hashmap
              // 如果内容过多会溢写到磁盘
              // 与上一篇的 insertAll 类似，这里就不展开了
              if (dep.mapSideCombine) {
                // We are reading values that are already combined
                val combinedKeyValuesIterator = interruptibleIter.asInstanceOf[Iterator[(K, C)]]
                dep.aggregator.get.combineCombinersByKey(combinedKeyValuesIterator, context)
              } else {
                // We don't know the value type, but also don't care -- the dependency *should*
                // have made sure its compatible w/ this aggregator, which will convert the value
                // type to the combined type C
                val keyValuesIterator = interruptibleIter.asInstanceOf[Iterator[(K, Nothing)]]
                dep.aggregator.get.combineValuesByKey(keyValuesIterator, context)
              }
            } else {
              require(!dep.mapSideCombine, "Map-side combine without Aggregator specified!")
              interruptibleIter.asInstanceOf[Iterator[Product2[K, C]]]
            }
        
            // 是否需要排序，如果不需要排序就可以直接返回了
            dep.keyOrdering match {
              case Some(keyOrd: Ordering[K]) =>
                // Create an ExternalSorter to sort the data. Note that if spark.shuffle.spill is disabled,
                // the ExternalSorter won't spill to disk.
                val sorter =
                  new ExternalSorter[K, C, C](context, ordering = Some(keyOrd), serializer = dep.serializer)
                // 又是一个上一篇见到的方法
                sorter.insertAll(aggregatedIter)
                context.taskMetrics().incMemoryBytesSpilled(sorter.memoryBytesSpilled)
                context.taskMetrics().incDiskBytesSpilled(sorter.diskBytesSpilled)
                context.taskMetrics().incPeakExecutionMemory(sorter.peakMemoryUsedBytes)
                CompletionIterator[Product2[K, C], Iterator[Product2[K, C]]](sorter.iterator, sorter.stop())
              case None =>
                aggregatedIter
            }
          }
    
7. 整个 read 流程已经走完了，下面简单看一下 read 阶段是怎么拉取数据的。在上面的代码中创建了一个 ShuffleBlockFetcherIterator，传入的参数有 spark.reducer.maxSizeInFlight (defult 48m) 代表每次能拉取的最大数据量。spark.reducer.maxReqsInFlight 代表远程主机在某一时刻能读取的最大 block 数量。在 ShuffleBlockFetcherIterator 的构造方法中调用了 initialize 方法，下面来看一下 initialize() 的内容。

          private[this] def initialize(): Unit = {
            context.addTaskCompletionListener(_ => cleanup())
        
            // 区分远程和本地的 block。返回远程 block 的 Requests
            val remoteRequests = splitLocalRemoteBlocks()
            // 打乱顺序...
            fetchRequests ++= Utils.randomize(remoteRequests)
            assert ((0 == reqsInFlight) == (0 == bytesInFlight),
              "expected reqsInFlight = 0 but found reqsInFlight = " + reqsInFlight +
              ", expected bytesInFlight = 0 but found bytesInFlight = " + bytesInFlight)
            // 向远端发送请求
            fetchUpToMaxBytes()
            val numFetches = remoteRequests.size - fetchRequests.size
            logInfo("Started " + numFetches + " remote fetches in" + Utils.getUsedTimeMs(startTime))
            // 本地..
            fetchLocalBlocks()
            logDebug("Got local blocks in " + Utils.getUsedTimeMs(startTime))
          }
    
    下面看 splitLocalRemoteBlocks 的具体内容：
    
          private[this] def splitLocalRemoteBlocks(): ArrayBuffer[FetchRequest] = {
            // 大于五分之一 maxBytesInFlight 的块可以作为一个单独的 FetchRequest
            val targetRequestSize = math.max(maxBytesInFlight / 5, 1L)
            logDebug("maxBytesInFlight: " + maxBytesInFlight + ", targetRequestSize: " + targetRequestSize)
            
            val remoteRequests = new ArrayBuffer[FetchRequest]
        
            var totalBlocks = 0
            for ((address, blockInfos) <- blocksByAddress) {
              totalBlocks += blockInfos.size
              // 过滤本地的 block
              if (address.executorId == blockManager.blockManagerId.executorId) {
                // Filter out zero-sized blocks
                localBlocks ++= blockInfos.filter(_._2 != 0).map(_._1)
                numBlocksToFetch += localBlocks.size
              } else {
                val iterator = blockInfos.iterator
                var curRequestSize = 0L
                var curBlocks = new ArrayBuffer[(BlockId, Long)]
                while (iterator.hasNext) {
                  val (blockId, size) = iterator.next()
                  // Skip empty blocks
                  if (size > 0) {
                    curBlocks += ((blockId, size))
                    remoteBlocks += blockId
                    numBlocksToFetch += 1
                    curRequestSize += size
                  } else if (size < 0) {
                    throw new BlockException(blockId, "Negative block size " + size)
                  }
                  // 当一个节点上的总 block size 大于 targetRequestSize
                  // 构成一个单独的 FetchRequest
                  if (curRequestSize >= targetRequestSize) {
                    // Add this FetchRequest
                    remoteRequests += new FetchRequest(address, curBlocks)
                    curBlocks = new ArrayBuffer[(BlockId, Long)]
                    logDebug(s"Creating fetch request of $curRequestSize at $address")
                    curRequestSize = 0
                  }
                }
                // 剩下的 blocks 构成一个 FetchRequest
                if (curBlocks.nonEmpty) {
                  remoteRequests += new FetchRequest(address, curBlocks)
                }
              }
            }
            logInfo(s"Getting $numBlocksToFetch non-empty blocks out of $totalBlocks blocks")
            remoteRequests
          }
    
    fetchUpToMaxBytes 方法向远端发送请求：
        
          private def fetchUpToMaxBytes(): Unit = {
            // 注意停止的条件 reqsInFlight 就是我们通过 spark.reducer.maxReqsInFlight 设置的值
            // 总的获取数据大小也要小于 maxBytesInFlight
            while (fetchRequests.nonEmpty &&
              (bytesInFlight == 0 ||
                (reqsInFlight + 1 <= maxReqsInFlight &&
                  bytesInFlight + fetchRequests.front.size <= maxBytesInFlight))) {
              // 这个方法再往下走就到了 netty 部分了
              sendRequest(fetchRequests.dequeue())
            }
          }
    


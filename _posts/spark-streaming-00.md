---
title: Spark源码-streaming-00-ReceiverTracker 数据接收
date: 2017-04-03 23:56:19
description: 
tags:
	- spark
---
## 问题
在开始了解 Streaming 架构之前，我有下面一些思考：

1. 如果让我自己来在 Spark RDD 的基础上封装 Streaming 应该怎么做。
2. 我会怎们处理 Streaming 的输入部分？ 是在 RDD 的基础上进一步封装？ 还是另起炉灶？
3. 如果另起炉灶，那么我接受到的数据应该怎么给后续的 RDD, 形成处理的流水线？
4. 如果我来设计，怎么保证我接收到的数据不管异常还是宕机都不会丢失？
5. 如果我依托于 RDD 实现 Streaming，我的数据接收部分与后续数据处理部分应该是什么样的关系？
6. 我的消息来不及处理产生堆积怎么办？
7. 我的设计方式会在哪些地方出问题？或者说不适用于什么样的场景？

我觉得在了解 Streaming 之前可以先自己想一想上面的问题, 然后对比一下 Steaming 是怎么做的，挺有意思的. 

## 总结
1. StreamingContext 使用 JobScheduler 负责总体调度
2. JobScheduler 使用 receiverTracker 启动 receiver job 负责接收输入的数据，jobGenerator 负责生成数据处理的 job
3. receiverTracker 实际上是提交了一个 Job 来启动 receiver 接收数据
4. receiverTracker 提交的 job 中使用 ReceiverSupervisorImpl 来负载数据接并管 block, 同时向 Driver 上的 receiverTracker 报告状态
5. ReceiverSupervisorImpl 使用 BlockGenerator 把接收的数据分块保存，spark.streaming.blockInterval 用于每个多少毫秒的数据保存成一个块
6. BlockGenerator 使用使用 Guava 的 RateLimiter 影响消息接收速率

## 流程
看下面的例子：

    val conf = new SparkConf().setAppName("spark-test").setMaster("local[2]")
    val ssc= new StreamingContext(conf,Seconds(1))
    val lines = ssc.socketTextStream("192.168.192.132", 9999)
    val words = lines.flatMap(_.split(" "))
    val pairs = words.map(word => (word, 1))
    val wordCounts = pairs.reduceByKey(_ + _)
    wordCounts.print()
    ssc.start()
    ssc.awaitTermination()

1. StreamingContext 对象创建，这里面做的比较重要的几件事情有：
    
    - 传入或则创建 SparkContext
    
          private[streaming] val sc: SparkContext = {
            if (_sc != null) {
              _sc
            } else if (isCheckpointPresent) {
              SparkContext.getOrCreate(_cp.createSparkConf())
            } else {
              throw new SparkException("Cannot create StreamingContext without a SparkContext")
            }
          }        

    - 创建 DStreamGraph, 这个类里面有 inputStreams 和 outputStreams 就是也就是输入和输出的 DStream. 比如上面例子中最后的 print 就会把自身的引用(ForEachDStream)注册到 DStreamGraph 的 outputStreams 里面. 同时这个类还用来生成 job, 后面会详细写到
    
          private[streaming] val graph: DStreamGraph = {
            if (isCheckpointPresent) {
              _cp.graph.setContext(this)
              _cp.graph.restoreCheckpointData()
              _cp.graph
            } else {
              require(_batchDur != null, "Batch duration for StreamingContext cannot be null")
              val newGraph = new DStreamGraph()
              newGraph.setBatchDuration(_batchDur)
              newGraph
            }
          }
    
    - 创建 JobScheduler 用于任务调度(内部 receiverTracker 负责 receiver 调度, jobGenerator 负责 job 生成)
        
          private[streaming] val scheduler = new JobScheduler(this)

2. 然后在 ssc.socketTextStream 创建 DStream 的时候返回的是一个 SocketInputDStream, 这个类是 InputDStream 的子类, 而在 InputDStream 中会把自己的引用注册到前面提到的 DStreamGraph 的 inputStreams 中. 下面来看 ssc.start() 方法

          def start(): Unit = synchronized {
            state match {
              // state 在 streamingContext 初始化时被设置为了 INITIALIZED 
              case INITIALIZED =>
                startSite.set(DStream.getCreationSite())
                StreamingContext.ACTIVATION_LOCK.synchronized {
                  StreamingContext.assertNoOtherContextIsActive()
                  try {
                    // 这个方法就是做了一些配置方面的检查，可以自己看看
                    validate()
        
                    // 这种写法应该很熟悉吧，科里化封装了线程启动的逻辑
                    // 下面会看一下 scheduler.start() 方法
                    ThreadUtils.runInNewThread("streaming-start") {
                      sparkContext.setCallSite(startSite.get)
                      sparkContext.clearJobGroup()
                      sparkContext.setLocalProperty(SparkContext.SPARK_JOB_INTERRUPT_ON_CANCEL, "false")
                      savedProperties.set(SerializationUtils.clone(sparkContext.localProperties.get()))
                      scheduler.start()
                    }
                    state = StreamingContextState.ACTIVE
                  } catch {
                    case NonFatal(e) =>
                      logError("Error starting the context, marking it as stopped", e)
                      scheduler.stop(false)
                      state = StreamingContextState.STOPPED
                      throw e
                  }
                  StreamingContext.setActiveContext(this)
                }
                logDebug("Adding shutdown hook")
                // 注册钩子函数，在虚拟机正常结束时会调用
                // 然后调用 stopOnShutdown 方法，最终调用
                // scheduler.stop(stopGracefully) ,重点是 stopGracefully
                // spark.streaming.stopGracefullyOnShutdown 可以是设置是否等到全部数据执行完再结束。
                // 默认是 false
                shutdownHookRef = ShutdownHookManager.addShutdownHook(
                  StreamingContext.SHUTDOWN_HOOK_PRIORITY)(stopOnShutdown)
                // Registering Streaming Metrics at the start of the StreamingContext
                assert(env.metricsSystem != null)
                env.metricsSystem.registerSource(streamingSource)
                uiTab.foreach(_.attach())
                logInfo("StreamingContext started")
              case ACTIVE =>
                logWarning("StreamingContext has already been started")
              case STOPPED =>
                throw new IllegalStateException("StreamingContext has already been stopped")
            }
          }

3. scheduler.start()

          def start(): Unit = synchronized {
            if (eventLoop != null) return // scheduler has already been started
            // 启动一个事件处理线程用于处理 Job 相关事件
            logDebug("Starting JobScheduler")
            eventLoop = new EventLoop[JobSchedulerEvent]("JobScheduler") {
              override protected def onReceive(event: JobSchedulerEvent): Unit = processEvent(event)
        
              override protected def onError(e: Throwable): Unit = reportError("Error in job scheduler", e)
            }
            eventLoop.start()
        
            // attach rate controllers of input streams to receive batch completion updates
            // 添加 inputDStream 的速率控制器到 streaming context
            // 用于在 job 完成后调节数据接收速率
            for {
              inputDStream <- ssc.graph.getInputStreams
              rateController <- inputDStream.rateController
            } ssc.addStreamingListener(rateController)
        
            listenerBus.start()
            // receiverTracker 是个比较重要的组件
            // 用于控制 InputDStream 的执行
            receiverTracker = new ReceiverTracker(ssc)
            inputInfoTracker = new InputInfoTracker(ssc)
        
            val executorAllocClient: ExecutorAllocationClient = ssc.sparkContext.schedulerBackend match {
              case b: ExecutorAllocationClient => b.asInstanceOf[ExecutorAllocationClient]
              case _ => null
            }
        
            executorAllocationManager = ExecutorAllocationManager.createIfEnabled(
              executorAllocClient,
              receiverTracker,
              ssc.conf,
              ssc.graph.batchDuration.milliseconds,
              clock)
            executorAllocationManager.foreach(ssc.addStreamingListener)
            // 下面看这段代码执行
            receiverTracker.start()
            jobGenerator.start()
            executorAllocationManager.foreach(_.start())
            logInfo("Started JobScheduler")
          }

4. receiverTracker.start() , 好吧其实这里面没有太多东西
        
          def start(): Unit = synchronized {
            if (isTrackerStarted) {
              throw new SparkException("ReceiverTracker already started")
            }
        
            if (!receiverInputStreams.isEmpty) {
              endpoint = ssc.env.rpcEnv.setupEndpoint(
                "ReceiverTracker", new ReceiverTrackerEndpoint(ssc.env.rpcEnv))

              // launchReceivers() 中发送了 StartAllReceivers 消息
              // StartAllReceivers 在 ReceiverTrackerEndpoint#receive 方法中
              if (!skipReceiverLaunch) launchReceivers()
              logInfo("ReceiverTracker started")
              trackerState = Started
            }
          }

5. 下面就是 Receivers 比较关键的部分了

        case StartAllReceivers(receivers) =>
          // 计算位置偏好
          // 里边会用到一个 preferredLocation 函数，但是我找不到具体实现在哪里 
          val scheduledLocations = schedulingPolicy.scheduleReceivers(receivers,      getExecutors)
          // 对每个 receivers 依次分配
          for (receiver <- receivers) {
            val executors = scheduledLocations(receiver.streamId)
            updateReceiverScheduledExecutors(receiver.streamId, executors)
            receiverPreferredLocations(receiver.streamId) =    receiver.preferredLocation
            // 下面来看 startReceiver 方法
            startReceiver(receiver, executors)
          }

6. startReceiver() 是一个很长的方法，可以分为下面几个重要部分：

    - startReceiverFunc 这个函数就像是 spark core 里面 action 操作中传入的函数，这个函数是要在 worker 上运行的。
        
            // Function to start the receiver on the worker node
            val startReceiverFunc: Iterator[Receiver[_]] => Unit =
            (iterator: Iterator[Receiver[_]]) => {
              if (!iterator.hasNext) {
                throw new SparkException(
                  "Could not start receiver as object not found.")
              }
              if (TaskContext.get().attemptNumber() == 0) {
                val receiver = iterator.next()
                assert(iterator.hasNext == false)
                val supervisor = new ReceiverSupervisorImpl(
                  receiver, SparkEnv.get, serializableHadoopConf.value, checkpointDirOption)
                supervisor.start()
                supervisor.awaitTermination()
              } else {
                // It's restarted by TaskScheduler, but we want to reschedule it again. So exit it.
              }
            }
    
    - 创建 RDD , 就是随便搞了个 RDD(ParallelCollectionRDD),貌似都没啥用处
    
          val receiverRDD: RDD[Receiver[_]] =
            if (scheduledLocations.isEmpty) {
              ssc.sc.makeRDD(Seq(receiver), 1)
            } else {
              val preferredLocations = scheduledLocations.map(_.toString).distinct
              ssc.sc.makeRDD(Seq(receiver -> preferredLocations))
            }
    
    - 提交 Job, 到这里就是纯正的 spark rdd 任务流程了
    
          ssc.sparkContext.setJobDescription(s"Streaming job running receiver $receiverId")
          ssc.sparkContext.setCallSite(Option(ssc.getStartSite()).getOrElse(Utils.getCallSite()))
    
          val future = ssc.sparkContext.submitJob[Receiver[_], Unit, Unit](
            receiverRDD, startReceiverFunc, Seq(0), (_, _) => Unit, ())

6. 具体 job 怎么提交怎么调度到 worker 上，然后又怎么执行起来就不细说了，下面看一下上面提到的 startReceiverFunc，主要就是如下几句：

        val receiver = iterator.next()
        assert(iterator.hasNext == false)
        val supervisor = new ReceiverSupervisorImpl(
          receiver, SparkEnv.get, serializableHadoopConf.value, checkpointDirOption)
        supervisor.start()
        supervisor.awaitTermination()

7. 在 ReceiverSupervisorImpl 的构造方法中几个比较重要的点是：
    
    - receivedBlockHandler, 根据配置不同有 WAL 和 普通类型

          private val receivedBlockHandler: ReceivedBlockHandler = {
            // 要使用预写日志需要：
            // 1. 调用StreamingContext的checkpoint()方法设置一个checkpoint目录
            // 2. spark.streaming.receiver.writeAheadLog.enable 参数设置为 true
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
    
    - ReceiverTrackerRef && ReceiverEndpoint
          
          // 向 ReceiverTracker 发送消息的对象
          private val trackerEndpoint = RpcUtils.makeDriverRef("ReceiverTracker", env.conf, env.rpcEnv)
          // 处理 ReceiverTracker 消息的监听
          private val endpoint = env.rpcEnv.setupEndpoint(
            "Receiver-" + streamId + "-" + System.currentTimeMillis(), new ThreadSafeRpcEndpoint {
              override val rpcEnv: RpcEnv = env.rpcEnv
        
              override def receive: PartialFunction[Any, Unit] = {
                case StopReceiver =>
                  logInfo("Received stop signal")
                  ReceiverSupervisorImpl.this.stop("Stopped by driver", None)
                case CleanupOldBlocks(threshTime) =>
                  logDebug("Received delete old batch signal")
                  cleanupOldBlocks(threshTime)
                case UpdateRateLimit(eps) =>
                  logInfo(s"Received a new rate limit: $eps.")
                  registeredBlockGenerators.asScala.foreach { bg =>
                    bg.updateRate(eps)
                  }
              }
            })
    
    - BlockGenerator
    
          private val registeredBlockGenerators = new ConcurrentLinkedQueue[BlockGenerator]()
        
          /** Divides received data records into data blocks for pushing in BlockManager. */
          private val defaultBlockGeneratorListener = new BlockGeneratorListener {
            def onAddData(data: Any, metadata: Any): Unit = { }
        
            def onGenerateBlock(blockId: StreamBlockId): Unit = { }
        
            def onError(message: String, throwable: Throwable) {
              reportError(message, throwable)
            }
        
            def onPushBlock(blockId: StreamBlockId, arrayBuffer: ArrayBuffer[_]) {
              pushArrayBuffer(arrayBuffer, None, Some(blockId))
            }
          }
          private val defaultBlockGenerator = createBlockGenerator(defaultBlockGeneratorListener)
    
8. 下面来看 supervisor.start()
    
          def start() {
            // 调用 BlockGenerator 的 start
            onStart()
            // 启动消息接收的服务，本例中是 SocketReceiver
            startReceiver()
          }
    
          override protected def onStart() {
            registeredBlockGenerators.asScala.foreach { _.start() }
          }
          
          /** Start receiver */
          def startReceiver(): Unit = synchronized {
            try {
              // onReceiverStart 是在跟 ReceiverTracker 通信
              // 注册 Receiver
              if (onReceiverStart()) {
                logInfo(s"Starting receiver $streamId")
                receiverState = Started
                receiver.onStart()
                logInfo(s"Called receiver $streamId onStart")
              } else {
                // The driver refused us
                stop("Registered unsuccessfully because Driver refused to start receiver " + streamId, None)
              }
            } catch {
              case NonFatal(t) =>
                stop("Error starting receiver " + streamId, Some(t))
            }
          }
          
          override protected def onReceiverStart(): Boolean = {
            val msg = RegisterReceiver(
              streamId, receiver.getClass.getSimpleName, host, executorId, endpoint)
            trackerEndpoint.askWithRetry[Boolean](msg)
          }

9. SocketReceiver 中的几个重要方法，可以看到就是 java 的 socket 编程

          def onStart() {
            logInfo(s"Connecting to $host:$port")
            try {
              socket = new Socket(host, port)
            } catch {
              case e: ConnectException =>
                restart(s"Error connecting to $host:$port", e)
                return
            }
            logInfo(s"Connected to $host:$port")
        
            // 启动新的线程，调用 receive
            // 到这里就可以接收消息了
            new Thread("Socket Receiver") {
              setDaemon(true)
              override def run() { receive() }
            }.start()
          }
          /** Create a socket connection and receive data until receiver is stopped */
          def receive() {
            try {
              // bytesToObjects 在本例中是 reciver 构造时传入的 bytesToLines
              val iterator = bytesToObjects(socket.getInputStream())
              while(!isStopped && iterator.hasNext) {
                // 存储，下面看这个方法
                store(iterator.next())
              }
              if (!isStopped()) {
                restart("Socket data stream had no more data")
              } else {
                logInfo("Stopped receiving")
              }
            } catch {
              case NonFatal(e) =>
                logWarning("Error receiving data", e)
                restart("Error receiving data", e)
            } finally {
              onStop()
            }
          }

10. store 方法最终会调用到 BlockGenerator#addData() 方法, 所以先来看看 BlockGenerator 的构造函数中做的事情：

    - 启动定时转储的线程，spark.streaming.blockInterval 每隔多少毫秒生成一个 block, 默认是 200 秒
    
          private val blockIntervalMs = conf.getTimeAsMs("spark.streaming.blockInterval", "200ms")
          require(blockIntervalMs > 0, s"'spark.streaming.blockInterval' should be a positive value")
          // RecurringTimer 会定期调用 updateCurrentBuffer 函数
          // 就是简单的循环，阻塞，然后再循环
          // 下面看一下 updateCurrentBuffer 方法
          private val blockIntervalTimer =
            new RecurringTimer(clock, blockIntervalMs, updateCurrentBuffer, "BlockGenerator")
            
          //  
          private def updateCurrentBuffer(time: Long): Unit = {
            try {
              var newBlock: Block = null
              synchronized {
                if (currentBuffer.nonEmpty) {
                  // 更新 currentBuffer 
                  val newBlockBuffer = currentBuffer
                  currentBuffer = new ArrayBuffer[Any]
                  val blockId = StreamBlockId(receiverId, time - blockIntervalMs)
                  listener.onGenerateBlock(blockId)
                  newBlock = new Block(blockId, newBlockBuffer)
                }
              }
              // 把需要生成 block 的数据放到队列里面
              if (newBlock != null) {
                blocksForPushing.put(newBlock)  // put is blocking when queue is full
              }
            } catch {
              case ie: InterruptedException =>
                logInfo("Block updating timer thread was interrupted")
              case e: Exception =>
                reportError("Error in block updating thread", e)
            }
          }
    
    - 实例化 blocksForPushing 队列，默认大小是 10
    
            private val blockQueueSize = conf.getInt("spark.streaming.blockQueueSize", 10)
            private val blocksForPushing = new ArrayBlockingQueue[Block](blockQueueSize)
    
    - blockPushingThread 用于运行 keepPushingBlocks 函数，循环把 blocksForPushing 中的数据消费掉
        
          private val blockPushingThread = new Thread() { override def run() {   keepPushingBlocks() } }
            
          private def keepPushingBlocks() {
            logInfo("Started block pushing thread")
        
            def areBlocksBeingGenerated: Boolean = synchronized {
              state != StoppedGeneratingBlocks
            }
        
            try {
              // 循环消费数据
              // pushBlock 最后调用 BlockGeneratorListener#onPushBlock 方法
              while (areBlocksBeingGenerated) {
                Option(blocksForPushing.poll(10, TimeUnit.MILLISECONDS)) match {
                  case Some(block) => pushBlock(block)
                  case None =>
                }
              }
        
              // 关闭时，保证先存储完队列中的数据
              logInfo("Pushing out the last " + blocksForPushing.size() + " blocks")
              while (!blocksForPushing.isEmpty) {
                val block = blocksForPushing.take()
                logDebug(s"Pushing block $block")
                pushBlock(block)
                logInfo("Blocks left to push " + blocksForPushing.size())
              }
              logInfo("Stopped block pushing thread")
            } catch {
              case ie: InterruptedException =>
                logInfo("Block pushing thread was interrupted")
              case e: Exception =>
                reportError("Error in block pushing thread", e)
            }
          }

11. pushBlock 会调用 BlockGeneratorListener#onPushBlock 方法，最终会调用 ReceiverSupervisorImpl#pushAndReportBlock 方法.

          def pushAndReportBlock(
              receivedBlock: ReceivedBlock,
              metadataOption: Option[Any],
              blockIdOption: Option[StreamBlockId]
            ) {
            val blockId = blockIdOption.getOrElse(nextBlockId)
            val time = System.currentTimeMillis
            // 这里的 receivedBlockHandler 底层就是交给 blockManager 来存储了
            val blockStoreResult = receivedBlockHandler.storeBlock(blockId, receivedBlock)
            logDebug(s"Pushed block $blockId in ${(System.currentTimeMillis - time)} ms")
            val numRecords = blockStoreResult.numRecords
            val blockInfo = ReceivedBlockInfo(streamId, numRecords, metadataOption, blockStoreResult)
            trackerEndpoint.askWithRetry[Boolean](AddBlock(blockInfo))
            logDebug(s"Reported block $blockId")
          }


12. 下面再来看看 BlockGenerator#addData 方法：

          def addData(data: Any): Unit = {
            if (state == Active) {
              // 此方法用于限流
              waitToPush()
              synchronized {
                if (state == Active) {
                  currentBuffer += data
                } else {
                  throw new SparkException(
                    "Cannot add data as BlockGenerator has not been started or has been stopped")
                }
              }
            } else {
              throw new SparkException(
                "Cannot add data as BlockGenerator has not been started or has been stopped")
            }
          }
    
    RateLimiter 的部分代码 
    
          private val maxRateLimit = conf.getLong("spark.streaming.receiver.maxRate", Long.MaxValue)
          // spark.streaming.backpressure.initialRate 参数控制
          // Guava 是 google 的工具类库(令牌桶算法)
          // 参考:
          // http://ifeve.com/guava-ratelimiter/  
          // http://xiaobaoqiu.github.io/blog/2015/07/02/ratelimiter/
          private lazy val rateLimiter = GuavaRateLimiter.create(getInitialRateLimit().toDouble)
          
          def waitToPush() {
            // 阻塞，直到拿到令牌返回
            rateLimiter.acquire()
          }
    
13. 从前面第 4 步开始到第 12 步都是在讲 receiverTracker.start()，下一篇博客开始看第三步的 jobGenerator.start() 部分.

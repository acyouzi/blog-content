---
title: Spark源码-Core-02-Driver启动和Executor调度
date: 2017-03-03 21:06:56
tags:
    - spark
---
## 总结
1. spark-submit 使用 client 模式启动会直接在本地启动 driver，在 worker 中启动调度启动 driver
2. --deploy-mode 选择为 cluster 时, 运行的任务 jar 要么放在分布式文件系统中，要么每个 worker 的相同位置放置一份。
3. driver 使用 DAGScheduler 划分 stage,使用 TaskSchedulerImpl 完成 task 调度
4. 重点是 master 对 executor 的资源调度，调度分为 spreadOut 模式和非 spreadOut 模式。
5. spreadOut 模式会尽量把 executor 分散到不同机器上，而非 spreadOut 模式会尽可能使资源集中。
6. 如果 spark.executor.cores 参数没有配置则 oneExecutorPerWorker 变量为 true，结果会导致本来应该在同一个 worker 上启动的多个 executor 变成了一个。也就是说不配置 spark.executor.cores 会导致同一个 app 的 executor 在每个 worker 上只能启动一个。
7. 如果没有配置 spark.executor.cores 则 minCoresPerExecutor 的默认大小是 1，也就是说默认每个 executor 分配一个核。
8. 一个 worker 上相同 worker 的 executor 只启动一个时，则在分配资源时对于内存只考虑一个 executor 的消耗，但是 cpu 会分配多个。
9. executor 的调度个人感觉比较重要，所以在代码中做了较多注释
10. 根据运行日志来看代码还是挺有用的，把日志级别设置为 debug, 可以把代码和日志输出互相验证。

## 提交流程
spark 的提交命令格式如下:

    bin/spark-submit \
       --class <main-class> \
       --master <master-url> \
       --deploy-mode <deploy-mode cluster|client> \
       --conf <key>=<value> \
       ... # other options
       <application-jar> \
       [application-arguments]
    
    // --deploy-mode: 是否把你的 driver 程序部署在 worker 节点上, 或者在本地作为一个额外的客户端 (default: client)
    // --master <master-url local| local[K]| spark://HOST:PORT| yarn>
    // local[K] 中 k 是运行的 worker 数量

spark-submit会运行 org.apache.spark.deploy.SparkSubmit

1. 下面将分析的集群运行在 Standalone 模式下的任务提交流程，--deploy-mode 分为两种模式 client 和 cluster 首先来看看一看这两种模式在代码中有什么不同,在 org.apache.spark.deploy.SparkSubmit 类的 prepareSubmitEnvironment 方法中能找到如下两段代码:

        // client mode 直接把我们的任务 jar 作为 mainClass
        // 在本地启动 driver
        if (deployMode == CLIENT) {
          childMainClass = args.mainClass
          if (isUserJar(args.primaryResource)) {
            childClasspath += args.primaryResource
          }
          if (args.jars != null) { childClasspath ++= args.jars.split(",") }
          if (args.childArgs != null) { childArgs ++= args.childArgs }
        }
        
        if (args.isStandaloneCluster) {
          if (args.useRest) {
            childMainClass = "org.apache.spark.deploy.rest.RestSubmissionClient"
            childArgs += (args.primaryResource, args.mainClass)
          } else {
            childMainClass = "org.apache.spark.deploy.Client"
            if (args.supervise) { childArgs += "--supervise" }
            Option(args.driverMemory).foreach { m => childArgs += ("--memory", m) }
            Option(args.driverCores).foreach { c => childArgs += ("--cores", c) }
            childArgs += "launch"
            childArgs += (args.master, args.primaryResource, args.mainClass)
          }
          if (args.childArgs != null) {
            childArgs ++= args.childArgs
          }
        }
    
    他们分别指定了不同的 mainClass 在 Cluster 模式下的提交方法有 Rest 和 legacy 两种，Rest 要更新一点。legacy 模式是通过启动一个 Client 类，然后向 Master 发送 RequestSubmitDriver 请求来完成。
    
    Master 在收到 RequestSubmitDriver 消息后会进行调度选择一个 Worker 发送 Driver 启动命令(LaunchDriver),然后 Worker 收到启动命令后首先需要把 jar 包拉取到本地的工作目录，所以如果使用 Cluster 模式要么你自己把 jar 包放到集群上每个 worker 节点的相同位置，要么放到 hdfs 这样的分布式存储系统中。
    
2. 现在假设使用的是 Standalone 集群，提交的部署模式为 CLIENT，这样直接启动运行的就是我们自己的程序了。在我们自己实现的代码中首先需要创建一个 org.apache.spark.SparkContext，下面简述对象创建做的几个关键事情：

        // 创建 并注册 job listener
        // 代码注释中有说 Listener bus is only used on the driver
        _jobProgressListener = new JobProgressListener(_conf)
        listenerBus.addListener(jobProgressListener)
        
        // 底层调用 org.apache.spark.SparkEnv#createDriverEnv
        _env = createSparkEnv(_conf, isLocal, listenerBus)
        SparkEnv.set(_env)
        
    这里面 jobProgressListener 看代码好像仅仅是在维护状态信息，当状态改变被消息通知改变状态信息，没有做其他事情。代码好乱

        // We need to register "HeartbeatReceiver" before "createTaskScheduler" because Executor will
        // retrieve "HeartbeatReceiver" in the constructor. (SPARK-6640)
        // HeartbeatReceiver 即使一个 RpcEndpoint 又是 SparkListener
        // HeartbeatReceiver 的构造函数中把自己注册到了 listenerBus 中
        // 这里注册到 rpcEnv 后会立即调用 OnStart 方法，在这个方法中启动了一个定期触发 ExpireDeadHosts 的线程。
        _heartbeatReceiver = env.rpcEnv.setupEndpoint(
          HeartbeatReceiver.ENDPOINT_NAME, new HeartbeatReceiver(this))
    
        // Create and start the scheduler
        val (sched, ts) = SparkContext.createTaskScheduler(this, master, deployMode)
        _schedulerBackend = sched
        _taskScheduler = ts
        // DAGScheduler 负责 stage 划分以及决定每个 stage 的最佳位置
        _dagScheduler = new DAGScheduler(this)
        _heartbeatReceiver.ask[Boolean](TaskSchedulerIsSet)
    
        // start TaskScheduler after taskScheduler sets DAGScheduler reference in DAGScheduler's
        // constructor
        _taskScheduler.start()
    
    在 SparkContext.createTaskScheduler 中实际执行的如下部分，返回 TaskSchedulerImpl 和 StandaloneSchedulerBackend 两个实例，其中 TaskSchedulerImpl 的功能大部分是在调用 StandaloneSchedulerBackend
    
          case SPARK_REGEX(sparkUrl) =>
            val scheduler = new TaskSchedulerImpl(sc)
            val masterUrls = sparkUrl.split(",").map("spark://" + _)
            val backend = new StandaloneSchedulerBackend(scheduler, sc, masterUrls)
            scheduler.initialize(backend)
            (backend, scheduler)
            
    _taskScheduler.start() 里面会调用 backend.start() ，然后创建 AppClient 完成任务注册。 
    
        val appDesc = new ApplicationDescription(sc.appName, maxCores, sc.executorMemory, command,
          appUIAddress, sc.eventLogDir, sc.eventLogCodec, coresPerExecutor, initialExecutorLimit)
        client = new StandaloneAppClient(sc.env.rpcEnv, masters, appDesc, this, conf)
        client.start()
        launcherBackend.setState(SparkAppHandle.State.SUBMITTED)
        waitForRegistration()
        launcherBackend.setState(SparkAppHandle.State.RUNNING)
    
    client.start() 方法做了如下事情:
        
        // 既然注册了那就去看看 onStart 方法中做了什么
        endpoint.set(rpcEnv.setupEndpoint("AppClient", new ClientEndpoint(rpcEnv)))
    
3. 在 ClientEndpoint onStart 方法中做的事情与 Worker 向 Master 注册时做的事情差不多，不会通过 zookeeper 获取哪个是活动主节点，会分别尝试我们配置的多个主节点，直到成功，向主节点发送的消息是 RegisterApplication，下面来看一看 主节点收到消息做了什么:
    
        case RegisterApplication(description, driver) =>
          // TODO Prevent repeated registrations from some driver
          if (state == RecoveryState.STANDBY) {
            // ignore, don't send response
          } else {
            logInfo("Registering app " + description.name)
            val app = createApplication(description, driver)
            registerApplication(app)
            logInfo("Registered app " + description.name + " with ID " + app.id)
            // 持久化到 zk
            persistenceEngine.addApplication(app)
            driver.send(RegisteredApplication(app.id, self))
            // 先不管他到底是怎么调度的
            schedule()
          }

4. 注册成功之后返回 RegisteredApplication 消息，注意这个时候 waitForRegistration 还在阻塞着, 直到 connected 调用的时候才会执行 registrationBarrier.release() 
    
         case RegisteredApplication(appId_, masterRef) =>
            appId.set(appId_)
            registered.set(true)
            master = Some(masterRef)
            listener.connected(appId.get)

    这里面仍然是各种属性修改，listener.connected(appId.get) 调用的是 launcherBackend.setAppId(appId) ，这个东西启动的时候的创建的 socket 地址是 InetAddress.getLoopbackAddress() ... 自己给自己发消息？

5. 到目前为止 App 的注册就已经完成了。虽然有还是有很多代码不太理解，很多逻辑感觉很奇怪，但是不管怎么说，注册已经完成了。剩下的内容就是 master 调度 worker 启动 executor, 以及 executor 向 driver 注册自己了。既然要向 driver 注册自己那么 driver 就需要先注册相应的 endpoint. 这个 endpoint 在前面第二步调用 backend.start() 向 master 注册 APP 之前就已经完成了。在 backend.start() 父类的 start() 方法中完成。
    
          driverEndpoint = createDriverEndpointRef(properties)

          protected def createDriverEndpointRef(
              properties: ArrayBuffer[(String, String)]): RpcEndpointRef = {
            rpcEnv.setupEndpoint(ENDPOINT_NAME, createDriverEndpoint(properties))
          }
          
          protected def createDriverEndpoint(properties: Seq[(String, String)]): DriverEndpoint = {
            new DriverEndpoint(rpcEnv, properties)
          }

    DriverEndpoint 就是后面用来接收处理 executor 消息的对象。

6. 现在再来看 master 的 schedule 方法, 因为前面注册进入的是 waitingApps 队列，所有 schedule 方法最后调用的 startExecutorsOnWorkers, 然后在方法中过滤出符合条件的worker然后调用具体的调度算法  scheduleExecutorsOnWorkers 函数. 函数中比较重要的是下面的代码：

        // 分配
        while (freeWorkers.nonEmpty) {
          freeWorkers.foreach { pos =>
            var keepScheduling = true
            // canLaunchExecutor 是看每个 worker 上剩余的资源还够不够
            // 如果一个worker只分配一个 Executor 则这个方法在第二次检查一个节点的分配
            // 只检查 cpu 核数，不会去检查内存还有其他
            while (keepScheduling && canLaunchExecutor(pos)) {
              coresToAssign -= minCoresPerExecutor
              assignedCores(pos) += minCoresPerExecutor
    
              // oneExecutorPerWorker 只分配一个 executor，在一个 worker 上
              // oneExecutorPerWorker 在我们没有设置 coresPerExecutor 时为 true
              // 例子: 三个 worker，每个 8 核，需要 4 个 executor, 每个 executor 4 个 cpu
              // 如果现在的情况是 spreadOutApps=false oneExecutorPerWorker=true
              // 实际上可能会变成有两个 executor 每个 8 core
              // 当然真正决定启动几个 executor 不是在这里
              // 这里只是处理 assignedExecutors
              // canLaunchExecutor(pos) 函数做判断时会用到 assignedExecutors 变量
              // 如果 coresPerExecutor.isEmpty 为空，也就是没有设置 spark.executor.cores 的情况下 oneExecutorPerWorker 为 true
              if (oneExecutorPerWorker) {
                assignedExecutors(pos) = 1
              } else {
                assignedExecutors(pos) += 1
              }
              
              // 这里如果 spreadOutApps 为 true 则 while 循环只会执行一次
              // 这样就可以把总需要的核分配到多个 worker 上
              // 如果 spreadOutApps 为 true 则会尽可能的几种分配到单台 worker上
              if (spreadOutApps) {
                keepScheduling = false
              }
            }
          }
          // 如果一趟 foreach 没有分配完
          // 过滤掉资源不够的，再来一次
          freeWorkers = freeWorkers.filter(canLaunchExecutor)
        }

7. 资源分配完成后，master 就该向 worker 发送启动命令了：

          private def allocateWorkerResourceToExecutors(
              app: ApplicationInfo,
              assignedCores: Int,
              coresPerExecutor: Option[Int],
              worker: WorkerInfo): Unit = {
            // 这里决定了在每个 worker 上启动几个 executor
            // 如果没有设置 coresPerExecutor 也就是 spark.executor.cores 就用这个值除以分配的核数.
            // 否则 .. 启动一个 executor ,得到所有的核
            // 这个地方与上一步的分配算法共同决定了资源的分配策略
            val numExecutors = coresPerExecutor.map { assignedCores / _ }.getOrElse(1)
            val coresToAssign = coresPerExecutor.getOrElse(assignedCores)
            for (i <- 1 to numExecutors) {
              val exec = app.addExecutor(worker, coresToAssign)
              launchExecutor(worker, exec)
              app.state = ApplicationState.RUNNING
            }
          }

8. master 启动函数做的事情有两件: 1. 把消息发送到 worker 2.通知 driver 给你分配了哪些资源，在哪里。

          private def launchExecutor(worker: WorkerInfo, exec: ExecutorDesc): Unit = {
            logInfo("Launching executor " + exec.fullId + " on worker " + worker.id)
            worker.addExecutor(exec)
            worker.endpoint.send(LaunchExecutor(masterUrl,
              exec.application.id, exec.id, exec.application.desc, exec.cores, exec.memory))
            exec.application.driver.send(
              ExecutorAdded(exec.id, worker.id, worker.hostPort, exec.cores, exec.memory))
          }

9. 然后就是 worker 收到启动消息，启动 executor，期间的过程实在没什么意思，最后的结果就是运行了下面的这条命令: 

        INFO ExecutorRunner: Launch command: 
        "/usr/local/jdk1.8.0_102/bin/java" 
            "-cp" "/usr/local/spark-2.1.0-bin-hadoop2.7/conf/:/usr/local/spark-2.1.0-bin-hadoop2.7/jars/*" 
            "-Xmx1024M" 
            "-Dspark.driver.port=46671" "org.apache.spark.executor.CoarseGrainedExecutorBackend" 
            "--driver-url" "spark://CoarseGrainedScheduler@192.168.192.132:46671"
            "--executor-id" "0" 
            "--hostname" "192.168.192.133" 
            "--cores" "1" 
            "--app-id" "app-20170303011253-0002" 
            "--worker-url" "spark://Worker@192.168.192.133:43756"

10. 下面直接看 Executor 启动做了什么事情

        // 去 driver 获取配置文件，这里对应的是前面第五步注册的那个类
        DriverEndpoint
        val driver = fetcher.setupEndpointRefByURI(driverUrl)
        val cfg = driver.askWithRetry[SparkAppConfig](RetrieveSparkAppConfig)
        val props = cfg.sparkProperties ++ Seq[(String, String)](("spark.app.id", appId))
        fetcher.shutdown()
        ....
        // 略
        ....
        
        // rpc 通信
        val env = SparkEnv.createExecutorEnv(
            driverConf, executorId, hostname, port, cores, cfg.ioEncryptionKey, isLocal = false)
        // 注册 executor 处理业务的关键类 CoarseGrainedExecutorBackend
        env.rpcEnv.setupEndpoint("Executor", new CoarseGrainedExecutorBackend(
            env.rpcEnv, driverUrl, executorId, hostname, cores, userClassPath, env))
        // WorkerWatcher 应该没做什么事情，好像就是在打日志
        workerUrl.foreach { url =>
            env.rpcEnv.setupEndpoint("WorkerWatcher", new WorkerWatcher(env.rpcEnv, url))
        }

11. 在 CoarseGrainedExecutorBackend 的 onStart 方法中会去向 driver 注册自己,注册完成后 driver 又给 executor 发回来了一个 RegisteredExecutor 消息。

        // 注册成功后实例化 Executor 
        // 这个类封装了一个线程池，用于运行任务
        case RegisteredExecutor =>
          logInfo("Successfully registered with driver")
          try {
            executor = new Executor(executorId, hostname, env, userClassPath, isLocal = false)
          } catch {
            case NonFatal(e) =>
              exitExecutor(1, "Unable to create executor due to " + e.getMessage, e)
          }

12. 到这里 driver 和 executor 的启动流程计算是完成了。然后需要分析的就是任务的调度流程了。
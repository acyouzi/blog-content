---
title: Spark源码-Core-03-stage划分&&task调度
date: 2017-03-13 20:00:51
tags:
    - spark
---
## 总结
断断续续看了一周多，终于...
1. stage 分为两类，ShuffleMapStage 和 ResultStage，最后一个 stage 是 ResultStage
2. stage 划分依据的就是 dependencies 的类型，也就是所说的宽依赖窄依赖。ShuffleDependency 划分 stage
3. 提交 stage 需要先提交父依赖缺失的 stage, 如果只是缺失某个 partition 则只运行缺失的 partition 对应的 task
3. 提交任务的方法叫 submitMissingTasks，为什么是 Miss, 因为提交任务只运行 stage 缺失的 task. 在第一次提交时 task 也就是 partition 运行的结果全部缺失。
4. Task 的数量就是 partitions 的数量，Task 根据 stage 的不同又分为 ShuffleMapTask和ResultTask
5. dagScheduler 负责 job 的 stage 划分，以及维护 stage 的完整性。
6. taskScheduler 负责 task 级别的调度，资源分配，本地化级别
7. CoarseGrainedSchedulerBackend 负责发送消息到 executor 的发送。
8. 本地化调度可以通过 spark.locality.wait，spark.locality.wait.process, spark.locality.wait.node, spark.locality.wait.rack 优化
9. 本地化调度通过适当加长等待时间可以避免网络 I/O 提高运行效率
10. 本地化调度的关键方法是 getAllowedLocalityLevel 方法，同时还要清楚 TaskSetManager 关于任务队列的处理，这个过程个人感觉很复杂，需要耐下心来慢慢读。

## 过程分析
1. spark rdd 的算子分为 Action , Transformation， 其中 Transformation 类型算子会触发 job 提交。

   Action 类型算子也可以再细分，分为宽依赖窄依赖两种，宽依赖的操作创建的 RDD 是 ShuffledRDD。

2. 触发 job 提交的算子最终会调用 org.apache.spark.SparkContext#runJob 方法，然后这里面转而去调用 dagScheduler.runJob 方法。然后经理一番辗转完成 stage 划分，创建 task 提交给 taskScheduler.submitTasks 负责去发送到各个 executor.走流程的代码非常无聊, 下面直接来看一些关键的代码：

        private[scheduler] def handleJobSubmitted(jobId: Int,
              finalRDD: RDD[_],
              func: (TaskContext, Iterator[_]) => _,
              partitions: Array[Int],
              callSite: CallSite,
              listener: JobListener,
              properties: Properties) {
            var finalStage: ResultStage = null
            try {
              // stage 划分
              finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite)
            } catch {
              case e: Exception =>
                logWarning("Creating new stage failed due to exception - job: " + jobId, e)
                listener.jobFailed(e)
                return
            }
            // 创建 job, 可以根据日志信息去跟源码，调试什么的就算了吧，莫名其妙的就跑飞了。
            val job = new ActiveJob(jobId, finalStage, callSite, listener, properties)
            clearCacheLocs()
            logInfo("Got job %s (%s) with %d output partitions".format(
              job.jobId, callSite.shortForm, partitions.length))
            logInfo("Final stage: " + finalStage + " (" + finalStage.name + ")")
            logInfo("Parents of final stage: " + finalStage.parents)
            logInfo("Missing parents: " + getMissingParentStages(finalStage))
        
            val jobSubmissionTime = clock.getTimeMillis()
            jobIdToActiveJob(jobId) = job
            activeJobs += job
            finalStage.setActiveJob(job)
            val stageIds = jobIdToStageIds(jobId).toArray
            val stageInfos = stageIds.flatMap(id => stageIdToStage.get(id).map(_.latestInfo))
            listenerBus.post(
              SparkListenerJobStart(job.jobId, jobSubmissionTime, stageInfos, properties))
            // stage 提交
            submitStage(finalStage)
        }

3. stage 划分是根据宽依赖窄依赖来划，createResultStage 里面首先 getOrCreateParentStages, 得到父类的 stage.

          private def getOrCreateParentStages(rdd: RDD[_], firstJobId: Int): List[Stage] = {
            getShuffleDependencies(rdd).map { shuffleDep =>
              getOrCreateShuffleMapStage(shuffleDep, firstJobId)
            }.toList
          }
    
    这里的 getShuffleDependencies 就是 stage 划分比较关键的地方了
    
          private[scheduler] def getShuffleDependencies(
          rdd: RDD[_]): HashSet[ShuffleDependency[_, _, _]] = {
            val parents = new HashSet[ShuffleDependency[_, _, _]]
            val visited = new HashSet[RDD[_]]
            val waitingForVisit = new Stack[RDD[_]]
            
            // 遍历返回 ShuffleDependency 类型的依赖
            waitingForVisit.push(rdd)
            while (waitingForVisit.nonEmpty) {
              val toVisit = waitingForVisit.pop()
              if (!visited(toVisit)) {
                visited += toVisit
                toVisit.dependencies.foreach {
                  // 判断类型 ...
                  case shuffleDep: ShuffleDependency[_, _, _] =>
                    parents += shuffleDep
                  case dependency =>
                    waitingForVisit.push(dependency.rdd)
                }
              }
            }
            parents
          }
    
    getOrCreateShuffleMapStage 中如果 stage 已经存在就直接返回，如果没有会先遍历创建父依赖的 ShuffleMapStage 然后再创建本节点的。

4. 下一步需要提交 stage, 会调用 submitStage，这里判断父依赖的 stage 是否是完整可用的，可用的判断条件是 

        def isAvailable: Boolean = _numAvailableOutputs == numPartitions
    
    getMissingParentStages 完成上面的判断，代码比较简单就不贴了。

          private def submitStage(stage: Stage) {
            val jobId = activeJobForStage(stage)
            if (jobId.isDefined) {
              logDebug("submitStage(" + stage + ")")
              if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {
                // 得到缺失 pattitions 的父类
                val missing = getMissingParentStages(stage).sortBy(_.id)
                logDebug("missing: " + missing)
                if (missing.isEmpty) {
                  logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
                  // 提交 task 
                  // 如果是已经运行过的 stage 重新运行则只会运行缺失的部分
                  // 如果是新的 stage 会运行全部的 partition
                  submitMissingTasks(stage, jobId.get)
                } else {
                  // 如果父类存在缺失，先提交父 stage.
                  // 最终调用的也是上面的 submitMissingTasks
                  for (parent <- missing) {
                    submitStage(parent)
                  }
                  waitingStages += stage
                }
              }
            } else {
              abortStage(stage, "No active job for stage " + stage.id, None)
            }
          }
    
    所以 dagScheduler 会保证 stage 结果完整，如果发现 stage 不完整会重试运行。
    
5. 下面看一下提交任务的 submitMissingTasks 方法，这是一个比较长的方法，这里边完成的工作有 1. 获取缺失的 partitions, 2. 计算 task 的 PreferredLocs 3. 序列化 rdd && broadcast 4. 创建 task 通过 taskScheduler.submitTasks 提交.

    计算 PreferredLocs 最终会调用 getPreferredLocsInternal 方法:

          private def getPreferredLocsInternal(
          rdd: RDD[_],
          partition: Int,
          visited: HashSet[(RDD[_], Int)]): Seq[TaskLocation] = {
            // If the partition has already been visited, no need to re-visit.
            // This avoids exponential path exploration.  SPARK-695
            if (!visited.add((rdd, partition))) {
              // Nil has already been returned for previously visited partitions.
              return Nil
            }
            // cache 缓存过的，直接返回缓存的位置 
            val cached = getCacheLocs(rdd)(partition)
            if (cached.nonEmpty) {
              return cached
            }
            // 好像跟 checkpoint 有点关系，不太懂，等看完 checkpoint 再说
            val rddPrefs = rdd.preferredLocations(rdd.partitions(partition)).toList
            if (rddPrefs.nonEmpty) {
              return rddPrefs.map(TaskLocation(_))
            }
        
            // 从窄依赖的 rdd 里面找，递归的找
            // 仍然是判断 cache 和 checkpoint
            rdd.dependencies.foreach {
              case n: NarrowDependency[_] =>
                for (inPart <- n.getParents(partition)) {
                  val locs = getPreferredLocsInternal(n.rdd, inPart, visited)
                  if (locs != Nil) {
                    return locs
                  }
                }
              case _ =>
            }
            Nil
          }
    
    序列化&&broadcast
        
        // 反序列化的时候每个 task 会有一个 RDD 实例，这样可以避免一些线程安全问题.
        var taskBinary: Broadcast[Array[Byte]] = null
        try {
          // For ShuffleMapTask, serialize and broadcast (rdd, shuffleDep).
          // For ResultTask, serialize and broadcast (rdd, func).
          val taskBinaryBytes: Array[Byte] = stage match {
            case stage: ShuffleMapStage =>
              JavaUtils.bufferToArray(
                closureSerializer.serialize((stage.rdd, stage.shuffleDep): AnyRef))
            case stage: ResultStage =>
              JavaUtils.bufferToArray(closureSerializer.serialize((stage.rdd, stage.func): AnyRef))
          }
    
          taskBinary = sc.broadcast(taskBinaryBytes)
        }
    
    创建 Task 根据 stage 不同创建不同的 task，这个 Task 会在 executor 上执行。
    
        val tasks: Seq[Task[_]] = try {
          stage match {
            case stage: ShuffleMapStage =>
              partitionsToCompute.map { id =>
                val locs = taskIdToLocations(id)
                val part = stage.rdd.partitions(id)
                new ShuffleMapTask(stage.id, stage.latestInfo.attemptId,
                  taskBinary, part, locs, stage.latestInfo.taskMetrics, properties, Option(jobId),
                  Option(sc.applicationId), sc.applicationAttemptId)
              }
    
            case stage: ResultStage =>
              partitionsToCompute.map { id =>
                val p: Int = stage.partitions(id)
                val part = stage.rdd.partitions(p)
                val locs = taskIdToLocations(id)
                new ResultTask(stage.id, stage.latestInfo.attemptId,
                  taskBinary, part, locs, id, properties, stage.latestInfo.taskMetrics,
                  Option(jobId), Option(sc.applicationId), sc.applicationAttemptId)
              }
          }
    
    然后调用 taskScheduler.submitTasks 把任务交给 taskScheduler 去处理。
    
        if (tasks.size > 0) {
          logInfo("Submitting " + tasks.size + " missing tasks from " + stage + " (" + stage.rdd + ")")
          stage.pendingPartitions ++= tasks.map(_.partitionId)
          logDebug("New pending partitions: " + stage.pendingPartitions)
          taskScheduler.submitTasks(new TaskSet(
            tasks.toArray, stage.id, stage.latestInfo.attemptId, jobId, properties))
          stage.latestInfo.submissionTime = Some(clock.getTimeMillis())
        }
    
    在 taskScheduler.submitTasks 中比较重要的一件事情是执行 createTaskSetManager 方法，在 TaskSetManager 的构造方法中做的一件事情是 addPendingTask 把 task 入队列，这件事情涉及到了后边的本地化级别调度。
        
          // 根据不同情况会进入不同的队列，这里工作的结果在后面计算本地化级别时会用到。
          private def addPendingTask(index: Int) {
            for (loc <- tasks(index).preferredLocations) {
              loc match {
                // 这里的 locations 就是前面 getPreferredLocsInternal 得到的 locations
                case e: ExecutorCacheTaskLocation =>
                  // 注意他进入了哪个队列
                  pendingTasksForExecutor.getOrElseUpdate(e.executorId, new ArrayBuffer) += index
                case e: HDFSCacheTaskLocation =>
                  // HDFS 的本地化 ... 计算位置靠近存储位置？
                  // 没仔细看
                  val exe = sched.getExecutorsAliveOnHost(loc.host)
                  exe match {
                    case Some(set) =>
                      for (e <- set) {
                        pendingTasksForExecutor.getOrElseUpdate(e, new ArrayBuffer) += index
                      }
                      logInfo(s"Pending task $index has a cached location at ${e.host} " +
                        ", where there are executors " + set.mkString(","))
                    case None => logDebug(s"Pending task $index has a cached location at ${e.host} " +
                        ", but there are no executors alive there.")
                  }
                case _ =>
              }
              pendingTasksForHost.getOrElseUpdate(loc.host, new ArrayBuffer) += index
              for (rack <- sched.getRackForHost(loc.host)) {
                pendingTasksForRack.getOrElseUpdate(rack, new ArrayBuffer) += index
              }
            }
            // 无偏好队列
            if (tasks(index).preferredLocations == Nil) {
              pendingTasksWithNoPrefs += index
            }
        
            allPendingTasks += index  // No point scanning this whole list to find the old task there
          }

6. taskScheduler 转而调用 backend.reviveOffers(), 然后在这个方法中发送了一个 ReviveOffers 给 driverEndpoint。最终处理是在 CoarseGrainedSchedulerBackend.DriverEndpoint#makeOffers 方法中。

        private def makeOffers() {
          // Filter out executors under killing
          val activeExecutors = executorDataMap.filterKeys(executorIsAlive)
          val workOffers = activeExecutors.map { case (id, executorData) =>
            new WorkerOffer(id, executorData.executorHost, executorData.freeCores)
          }.toIndexedSeq
          launchTasks(scheduler.resourceOffers(workOffers))
        }

    首先调用 scheduler.resourceOffers 进行资源分配，然后 launchTasks,  resourceOffers 是一个超级大的方法，在这个方法里面做的第一件事情是检查 host,executor 信息。

        var newExecAvail = false
        for (o <- offers) {
          // 检查主机
          if (!hostToExecutors.contains(o.host)) {
            hostToExecutors(o.host) = new HashSet[String]()
          }
          // 检查 executor 
          if (!executorIdToRunningTaskIds.contains(o.executorId)) {
            hostToExecutors(o.host) += o.executorId
            executorAdded(o.executorId, o.host)
            executorIdToHost(o.executorId) = o.host
            executorIdToRunningTaskIds(o.executorId) = HashSet[Long]()
            newExecAvail = true
          }
          // 机架信息，TaskSchedulerImpl 里面直接返回 Null
          for (rack <- getRackForHost(o.host)) {
            hostsByRack.getOrElseUpdate(rack, new HashSet[String]()) += o.host
          }
        }
    
    第二步是进行资源分配, 下面是具体代码
    
        // 打乱顺序，均衡负载
        val shuffledOffers = Random.shuffle(offers)
        // 记录，会返回给上一层
        val tasks = shuffledOffers.map(o => new ArrayBuffer[TaskDescription](o.cores))
        // 每个 executor 空闲的 core 用作资源分配
        val availableCpus = shuffledOffers.map(o => o.cores).toArray
        val sortedTaskSets = rootPool.getSortedTaskSetQueue
        for (taskSet <- sortedTaskSets) {
          logDebug("parentName: %s, name: %s, runningTasks: %s".format(
            taskSet.parent.name, taskSet.name, taskSet.runningTasks))
          // 如果有新的 executor ， 从新分配资源
          if (newExecAvail) {
            taskSet.executorAdded()
          }
        }
    
        // 资源分配
        // 按照本地化级别调度分配资源
        // PROCESS_LOCAL( 在同一个 executor 中 )
        // NODE_LOCAL( 在同一个节点中 )
        // NO_PREF( 无偏好 )
        // RACK_LOCAL( 在同一个机架中 )
        // ANY( ... ) 
        for (taskSet <- sortedTaskSets) {
          var launchedAnyTask = false
          var launchedTaskAtCurrentMaxLocality = false
          // 计算本地化级别，在各个级别分配资源
          for (currentMaxLocality <- taskSet.myLocalityLevels) {
            do {
              // 单个节点的资源分配
              launchedTaskAtCurrentMaxLocality = resourceOfferSingleTaskSet(
                taskSet, currentMaxLocality, shuffledOffers, availableCpus, tasks)
              launchedAnyTask |= launchedTaskAtCurrentMaxLocality
            } while (launchedTaskAtCurrentMaxLocality)
          }
          if (!launchedAnyTask) {
            taskSet.abortIfCompletelyBlacklisted(hostToExecutors)
          }
        }

    下面首先来看一看本地化级别的计算 myLocalityLevels

          private def computeValidLocalityLevels(): Array[TaskLocality.TaskLocality] = {
            import TaskLocality.{PROCESS_LOCAL, NODE_LOCAL, NO_PREF, RACK_LOCAL, ANY}
            val levels = new ArrayBuffer[TaskLocality.TaskLocality]
            
            // 如果 pendingTasksForExecutor 不为空并且 PROCESS_LOCAL 设置不为 0
            // 这个的 pendingTasksForExecutor 就是前面在创建 TaskSetManager 是添加的队列
            // 这也就是位置偏好跟本地化级别形成关联的地方。
            if (!pendingTasksForExecutor.isEmpty && getLocalityWait(PROCESS_LOCAL) != 0 &&
                pendingTasksForExecutor.keySet.exists(sched.isExecutorAlive(_))) {
              levels += PROCESS_LOCAL
            }
            if (!pendingTasksForHost.isEmpty && getLocalityWait(NODE_LOCAL) != 0 &&
                pendingTasksForHost.keySet.exists(sched.hasExecutorsAliveOnHost(_))) {
              levels += NODE_LOCAL
            }
            if (!pendingTasksWithNoPrefs.isEmpty) {
              levels += NO_PREF
            }
            if (!pendingTasksForRack.isEmpty && getLocalityWait(RACK_LOCAL) != 0 &&
                pendingTasksForRack.keySet.exists(sched.hasHostAliveOnRack(_))) {
              levels += RACK_LOCAL
            }
            levels += ANY
            logDebug("Valid locality levels for " + taskSet + ": " + levels.mkString(", "))
            levels.toArray
          }
    
    这里面调用的 getLocalityWait 是根据我们自己的配置来返回值, 从下面的方法中可以看到我们能够对多种本地化级别的等待时间进行配置。

          private def getLocalityWait(level: TaskLocality.TaskLocality): Long = {
            val defaultWait = conf.get("spark.locality.wait", "3s")
            val localityWaitKey = level match {
              case TaskLocality.PROCESS_LOCAL => "spark.locality.wait.process"
              case TaskLocality.NODE_LOCAL => "spark.locality.wait.node"
              case TaskLocality.RACK_LOCAL => "spark.locality.wait.rack"
              case _ => null
            }
            if (localityWaitKey != null) {
              conf.getTimeAsMs(localityWaitKey, defaultWait)
            } else {
              0L
            }
          }
    
7. 根据前面的代码能了解到本地化级别是怎么计算出来的，以及他与位置偏好之间是怎么形成关系的。下面再来看看根据不同的本地化级别，采取了怎样的资源分配方式，参看 resourceOfferSingleTaskSet 方法，这个方法里面真正在做事情的是 resourceOffer 方法，其余的代码特别简单就不贴出来了，重点看 taskSet.resourceOffer(execId, host, maxLocality) 方法.
    
    这是一个比较长的方法，我把方法分成三部分，第一部分是确定本地化级别，代码如下

        var allowedLocality = maxLocality
        if (maxLocality != TaskLocality.NO_PREF) {
            allowedLocality = getAllowedLocalityLevel(curTime)
            if (allowedLocality > maxLocality) {
              // We're not allowed to search for farther-away tasks
              allowedLocality = maxLocality
            }
        }
    
    代码中通过 getAllowedLocalityLevel 确定本地化级别，也是在这个方法中完成了我寻找已久的本地化调度策略。代码如下：
    
         private def getAllowedLocalityLevel(curTime: Long): TaskLocality.TaskLocality = {
            // 略去几个无关紧要的方法
            ...
            ...
            ...
            
            // currentLocalityIndex 从 0 开始，也就是从 PROCESS_LOCAL 开始
            while (currentLocalityIndex < myLocalityLevels.length - 1) {
              // 检测当前级别有没有任务需要调度
              val moreTasks = myLocalityLevels(currentLocalityIndex) match {
                case TaskLocality.PROCESS_LOCAL => moreTasksToRunIn(pendingTasksForExecutor)
                case TaskLocality.NODE_LOCAL => moreTasksToRunIn(pendingTasksForHost)
                case TaskLocality.NO_PREF => pendingTasksWithNoPrefs.nonEmpty
                case TaskLocality.RACK_LOCAL => moreTasksToRunIn(pendingTasksForRack)
              }
              if (!moreTasks) {
                // 如果当前级别没有任务需要调度直接进入下一级别
                lastLaunchTime = curTime
                logDebug(s"No tasks for locality level ${myLocalityLevels(currentLocalityIndex)}, " +
                  s"so moving to locality level ${myLocalityLevels(currentLocalityIndex + 1)}")
                currentLocalityIndex += 1
              } else if (curTime - lastLaunchTime >= localityWaits(currentLocalityIndex)) {
                // 如果当前级别有任务需要调度，但是本次调度时间距离上次调度时间已经超过我们设置的本地化等待时间
                // 则进入下一个级别
                // 注意这里就是本地化调度的关键所在
                lastLaunchTime += localityWaits(currentLocalityIndex)
                logDebug(s"Moving to ${myLocalityLevels(currentLocalityIndex + 1)} after waiting for " +
                  s"${localityWaits(currentLocalityIndex)}ms")
                currentLocalityIndex += 1
              } else {
                // 如果两次调度的间隔时间不超过等待时间
                // 仍然以当前级别执行调度
                return myLocalityLevels(currentLocalityIndex)
              }
            }
            myLocalityLevels(currentLocalityIndex)
          }
    
    taskSet.resourceOffer 方法的第二部分是调用一个 dequeueTask(execId, host, allowedLocality) 方法去获取任务。调用时传入前边得到的本地化级别。
    
         private def dequeueTask(execId: String, host: String, maxLocality: TaskLocality.Value)
        : Option[(Int, TaskLocality.Value, Boolean)] =
          {
            // 因为进程本地化是最小的级别，如果这里面还有任务的是一定不会轮到后面的级别调度的。所以这里没有做过多的检测
            for (index <- dequeueTaskFromList(execId, host, getPendingTasksForExecutor(execId))) {
              return Some((index, TaskLocality.PROCESS_LOCAL, false))
            }
            // isAllowed 判断传入的调度级别是不是大于等于 TaskLocality.NODE_LOCAL
            // 如果是，就可以从 getPendingTasksForHost 中获取任务。
            // 后面的代码也是一样的过程，如果上一级还有调度的任务就轮不到下一级调度
            if (TaskLocality.isAllowed(maxLocality, TaskLocality.NODE_LOCAL)) {
              for (index <- dequeueTaskFromList(execId, host, getPendingTasksForHost(host))) {
                return Some((index, TaskLocality.NODE_LOCAL, false))
              }
            }
        
            if (TaskLocality.isAllowed(maxLocality, TaskLocality.NO_PREF)) {
              // Look for noPref tasks after NODE_LOCAL for minimize cross-rack traffic
              for (index <- dequeueTaskFromList(execId, host, pendingTasksWithNoPrefs)) {
                return Some((index, TaskLocality.PROCESS_LOCAL, false))
              }
            }
        
            if (TaskLocality.isAllowed(maxLocality, TaskLocality.RACK_LOCAL)) {
              for {
                rack <- sched.getRackForHost(host)
                index <- dequeueTaskFromList(execId, host, getPendingTasksForRack(rack))
              } {
                return Some((index, TaskLocality.RACK_LOCAL, false))
              }
            }
        
            if (TaskLocality.isAllowed(maxLocality, TaskLocality.ANY)) {
              for (index <- dequeueTaskFromList(execId, host, allPendingTasks)) {
                return Some((index, TaskLocality.ANY, false))
              }
            }
        
            // 最后这个有点特别，应该是调度那些运行时间过长的任务的
            // 因为是在看任务提交流程，这里就贴详细代码了，简单介绍一些
            // 这个方法从 speculatableTasks(HashSet) 中拿数据
            // 而这个 hashset 里面的数据是通过定期调用 TaskSchedulerImpl#checkSpeculatableTasks 
            // 方法得到的，每当调用 checkSpeculatableTasks 时也会顺便调用 backend.reviveOffers() 发送 ReviveOffers 消息。
            // 至于定期调度的实现，是在一开始 TaskSchedulerImpl 启动的 start 方法中
            // 同时启动了一个任务线程定期调度。
            dequeueSpeculativeTask(execId, host, maxLocality).map {
              case (taskIndex, allowedLocality) => (taskIndex, allowedLocality, true)}
          }
    
    获取到 task 后的第三步是各种变量更新，包装 TaskDescription 返回。到这里任务分配工作就完成了。下面需要进行的就是通知 exector 启动 task 了。

8. 发送消息启动 task 消息的并没有做太多事情，只是把消息序列化然后发送到对应 executor 上。

        private def launchTasks(tasks: Seq[Seq[TaskDescription]]) {
          for (task <- tasks.flatten) {
            val serializedTask = ser.serialize(task)
            if (serializedTask.limit >= maxRpcMessageSize) {
              scheduler.taskIdToTaskSetManager.get(task.taskId).foreach { taskSetMgr =>
                try {
                  var msg = "Serialized task %s:%d was %d bytes, which exceeds max allowed: " +
                    "spark.rpc.message.maxSize (%d bytes). Consider increasing " +
                    "spark.rpc.message.maxSize or using broadcast variables for large values."
                  msg = msg.format(task.taskId, task.index, serializedTask.limit, maxRpcMessageSize)
                  taskSetMgr.abort(msg)
                } catch {
                  case e: Exception => logError("Exception in error callback", e)
                }
              }
            }
            else {
              val executorData = executorDataMap(task.executorId)
              executorData.freeCores -= scheduler.CPUS_PER_TASK
    
              logDebug(s"Launching task ${task.taskId} on executor id: ${task.executorId} hostname: " +
                s"${executorData.executorHost}.")
    
              executorData.executorEndpoint.send(LaunchTask(new SerializableBuffer(serializedTask)))
            }
          }
        }

9. 下面需要看的是 task 在 executor 上怎么运行。
---
title: Spark源码-Core-01-Worker注册流程
date: 2017-03-01 10:56:02
tags:
    - spark
---
## 简介
worker 的启动流程与 master 并无二致，这里不做过多介绍。下面先来总结一下 worker 注册的相关工作。
1. 如果采用 ha 模式部署集群，worker 不会通过询问 zookeeper 的方式获取 master.而是挨个尝试。
2. worker 如果注册不成功会不断尝试注册，当尝试超过一定次数会加大尝试间隔。
3. 注册成功后 worker 还需要定期向 master 发送心跳，心跳间隔是四分之一的超时时间。
4. 因为id 中包含时间，worker 注册的 id 在不同批次的尝试中是不相同的

## 注册流程
1. 在 OnStart 方法中调用 registerWithMaster() 方法发送注册消息

          private def registerWithMaster() {
            // onDisconnected may be triggered multiple times, so don't attempt registration
            // if there are outstanding registration attempts scheduled.
            registrationRetryTimer match {
              case None =>
                registered = false
                registerMasterFutures = tryRegisterAllMasters()
                connectionAttemptCount = 0
                registrationRetryTimer = Some(forwordMessageScheduler.scheduleAtFixedRate(
                  new Runnable {
                    override def run(): Unit = Utils.tryLogNonFatalError {
                      // 重点看这里
                      // 1. self 得到的是 worker endpoint 的引用
                      // 2. Option(self) 封装说明结果有可能为 null
                      // 3. Option 的 apply 方法会对传入的东西判断是否为 null 决定返回的是 None 还是 Some
                      // 4. Option 的 foreach 针对空值自动略过
                      // 5. 最后实际上调用了 workerendpointRef.send 也就是发消息给自己
                      Option(self).foreach(_.send(ReregisterWithMaster))
                    }
                  },
                  INITIAL_REGISTRATION_RETRY_INTERVAL_SECONDS,
                  INITIAL_REGISTRATION_RETRY_INTERVAL_SECONDS,
                  TimeUnit.SECONDS))
              case Some(_) =>
                logInfo("Not spawning another attempt to register with the master, since there is an" +
                  " attempt scheduled already.")
            }
          }

2. 上面发送的消息 ReregisterWithMaster 最终处理是在 receive 方法的 case ReregisterWithMaster 中调用 reregisterWithMaster() 方法
    
          private def reregisterWithMaster(): Unit = {
            // 捕获错误，不让抛出的异常导致程序停止。
            Utils.tryOrExit {
              connectionAttemptCount += 1
              // 已经注册成功了，则直接取消所有正在尝试注册的任务然后直接退出
              if (registered) {
                cancelLastRegistrationRetry()
              } else if (connectionAttemptCount <= TOTAL_REGISTRATION_RETRIES) {
                // 根据条件，如果没有达到最大尝试次数，就会接着尝试
                logInfo(s"Retrying connection to master (attempt # $connectionAttemptCount)")
                /**
                 * 这里本来有一大堆注释，说的是一个因为没有及时取消上一轮的注册尝试导致的重复注册错误。
                 */
                master match {
                  // 第一次注册肯定不会进入这个 case，因为 master 为空(None)。
                  // 注册成功后会更新 master 为我们注册的 master 的 ref
                  // 因为 我们可能配置有多个 master 节点，这里面第一个 case 只是在尝试重连曾经注册成功过的那个。
                  case Some(masterRef) =>
                    // registered == false && master != None means we lost the connection to master, so
                    // masterRef cannot be used and we need to recreate it again. Note: we must not set
                    // master to None due to the above comments.
                    if (registerMasterFutures != null) {
                      registerMasterFutures.foreach(_.cancel(true))
                    }
                    val masterAddress = masterRef.address
                    registerMasterFutures = Array(registerMasterThreadPool.submit(new Runnable {
                      override def run(): Unit = {
                        try {
                          logInfo("Connecting to master " + masterAddress + "...")
                          val masterEndpoint = rpcEnv.setupEndpointRef(masterAddress, Master.ENDPOINT_NAME)
                          registerWithMaster(masterEndpoint)
                        } catch {
                          case ie: InterruptedException => // Cancelled
                          case NonFatal(e) => logWarning(s"Failed to connect to master $masterAddress", e)
                        }
                      }
                    }))
                  case None =>
                    // 两种到达这个 case 的原因， 
                    // 1. 首次注册 2.master 挂了
                    if (registerMasterFutures != null) {
                      registerMasterFutures.foreach(_.cancel(true))
                    }
                    // We are retrying the initial registration
                    registerMasterFutures = tryRegisterAllMasters()
                }
                
                // 这里是超过一定阈值，然后尝试间隔加大
                if (connectionAttemptCount == INITIAL_REGISTRATION_RETRIES) {
                  // 更新发送 ReregisterWithMaster 线程的调度时间。
                  // 前面 onstart 方法中设置过一次
                  registrationRetryTimer.foreach(_.cancel(true))
                  registrationRetryTimer = Some(
                    forwordMessageScheduler.scheduleAtFixedRate(new Runnable {
                      override def run(): Unit = Utils.tryLogNonFatalError {
                        self.send(ReregisterWithMaster)
                      }
                    }, PROLONGED_REGISTRATION_RETRY_INTERVAL_SECONDS,
                      PROLONGED_REGISTRATION_RETRY_INTERVAL_SECONDS,
                      TimeUnit.SECONDS))
                }
              } else {
                logError("All masters are unresponsive! Giving up.")
                System.exit(1)
              }
            }
          }

3. 当启动 worker 时，首次注册调用了 tryRegisterAllMasters() :
          
          private def tryRegisterAllMasters(): Array[JFuture[_]] = {
            // 可能配置有多个 master 节点。都要尝试连接。
            masterRpcAddresses.map { masterAddress =>
              registerMasterThreadPool.submit(new Runnable {
                override def run(): Unit = {
                  try {
                    logInfo("Connecting to master " + masterAddress + "...")
                    // 通过地址获取远程节点的引用。
                    val masterEndpoint = rpcEnv.setupEndpointRef(masterAddress, Master.ENDPOINT_NAME)
                    registerWithMaster(masterEndpoint)
                  } catch {
                    case ie: InterruptedException => // Cancelled
                    case NonFatal(e) => logWarning(s"Failed to connect to master $masterAddress", e)
                  }
                }
              })
            }
          }

4. 最终调用 registerWithMaster(endpointref) 方法,使用 endpoint.ask 发送注册消息到 master.

          private def registerWithMaster(masterEndpoint: RpcEndpointRef): Unit = {
            // ask 最终底层使用的是 netty 的 client.sendRpc(content, this) 通过 netty
            masterEndpoint.ask[RegisterWorkerResponse](RegisterWorker(
              workerId, host, port, self, cores, memory, workerWebUiUrl))
              .onComplete {
                // This is a very fast action so we can use "ThreadUtils.sameThread"
                case Success(msg) =>
                  Utils.tryLogNonFatalError {
                    // 成功的时候调用 handleRegisterResponse 修改各种状态。
                    handleRegisterResponse(msg)
                  }
                case Failure(e) =>
                  logError(s"Cannot register with master: ${masterEndpoint.address}", e)
                  System.exit(1)
              }(ThreadUtils.sameThread)
          }

5. 注册成功之做的一些操作如下:

          case RegisteredWorker(masterRef, masterWebUiUrl) =>
            logInfo("Successfully registered with master " + masterRef.address.toSparkURL)
            // 标志位
            registered = true
            // master 成员变量修改为 masterRef，通过 registerMasterFutures 取消其余正在尝试的注册的任务
            changeMaster(masterRef, masterWebUiUrl)
            // 心跳，默认时间间隔是 四分之一的超时时间。
            // HEARTBEAT_MILLIS = conf.getLong("spark.worker.timeout", 60) * 1000 / 4
            forwordMessageScheduler.scheduleAtFixedRate(new Runnable {
              override def run(): Unit = Utils.tryLogNonFatalError {
                self.send(SendHeartbeat)
              }
            }, 0, HEARTBEAT_MILLIS, TimeUnit.MILLISECONDS)
            if (CLEANUP_ENABLED) {
              logInfo(
                s"Worker cleanup enabled; old application directories will be deleted in: $workDir")
              // 清理相关任务调度
              forwordMessageScheduler.scheduleAtFixedRate(new Runnable {
                override def run(): Unit = Utils.tryLogNonFatalError {
                  self.send(WorkDirCleanup)
                }
              }, CLEANUP_INTERVAL_MILLIS, CLEANUP_INTERVAL_MILLIS, TimeUnit.MILLISECONDS)
            }
            // 向 master 汇报 executor 信息
            val execs = executors.values.map { e =>
              new ExecutorDescription(e.appId, e.execId, e.cores, e.state)
            }
            masterRef.send(WorkerLatestState(workerId, execs.toList, drivers.keys.toSeq))

6. 到这里 worker 的注册部分就完成了，下面来看一看当 worker 注册时, master 做了哪些处理

          case RegisterWorker(
            id, workerHost, workerPort, workerRef, cores, memory, workerWebUiUrl) =>
          logInfo("Registering worker %s:%d with %d cores, %s RAM".format(
            workerHost, workerPort, cores, Utils.megabytesToString(memory)))
          // standby 节点不接受注册
          // 不能重复注册
          if (state == RecoveryState.STANDBY) {
            context.reply(MasterInStandby)
            // worker 的 id 生成通过如下方法
            // "worker-%s-%s-%d".format(createDateFormat.format(new Date), host, port)
            // 所以这里过滤的是同一批次尝试中的不同请求之间的重复注册。
            // 可能还存在id 不同 hostip port 相同的 worker 引用，要在 register 中清除。
          } else if (idToWorker.contains(id)) {
            context.reply(RegisterWorkerFailed("Duplicate worker ID"))
          } else {
            val worker = new WorkerInfo(id, workerHost, workerPort, cores, memory,
              workerRef, workerWebUiUrl)
            // registerWorker 首先要完成一些清理工作，然后注册节点
            if (registerWorker(worker)) {
              // 持久化，如果使用 zookeeper 做 ha 那么这个持久化引擎是在往 zookeeper 上同步数据。
              // 根据选择的模式不同，持久化引擎也不同
              persistenceEngine.addWorker(worker)
              // 回复结果
              context.reply(RegisteredWorker(self, masterWebUiUrl))
              // 这个方法是 spark 中最重要的调度算法，
              // 在每一次新的应用加入或可用资源发生改变时都会调用
              schedule()
            } else {
              val workerAddress = worker.endpoint.address
              logWarning("Worker registration failed. Attempted to re-register worker at same " +
                "address: " + workerAddress)
              context.reply(RegisterWorkerFailed("Attempted to re-register worker at same address: "
                + workerAddress))
            }
          }

7. 接下来除了任务调度 worker 和 master 之间需要做的事情就是保持心跳了，worker 向 master 发送完心跳请求，master 会更新 worker 的最后一次心跳时间。

          case Heartbeat(workerId, worker) =>
          idToWorker.get(workerId) match {
            case Some(workerInfo) =>
              workerInfo.lastHeartbeat = System.currentTimeMillis()
            case None =>
              // 如果收到一个不在列表里面的心跳，可能是心跳超时导致节点被清理了
              // 需要节点重新注册。
              if (workers.map(_.id).contains(workerId)) {
                logWarning(s"Got heartbeat from unregistered worker $workerId." +
                  " Asking it to re-register.")
                worker.send(ReconnectWorker(masterUrl))
              } else {
                logWarning(s"Got heartbeat from unregistered worker $workerId." +
                  " This worker was never registered, so ignoring the heartbeat.")
              }
          }

    在 master 启动里讲到过在 onStart 方法中会启动一个单独的线程去定期调度 CheckForWorkerTimeOut ，这里面判断心跳超时的方法是:

        val toRemove = workers.filter(_.lastHeartbeat < currentTime - WORKER_TIMEOUT_MS).toArray
    
    也就是超过四个心跳时间没有收到心跳包就认为 worker 超时。

8. 到这里 worker 注册相关的流程就结束了。


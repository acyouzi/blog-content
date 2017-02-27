---
title: Spark源码阅读笔记-Core-00-Master启动流程
date: 2017-02-27 22:10:33
tags:
    - spark
---
## 简介
其实这部分应该算是总结吧，也是读完 master 启动部分代码的一些理解:

1. endpoint 是用来处理 dispatcher 分发来的请求的, master 也是一个 endpoint
2. dispatcher 在一个单独的线程池中，也就是消息处理的逻辑在一个单独的线程池中执行。
3. dispatcher 的消息来源有两类，一类是 netty 通过网络接受来的数据, 另一类是本通过 masterEndpointRef 发送来的数据
4. netty 启动时在 pipeline 中注册的处理器有一个是 TransportChannelHandler 这个类封装了 TransportRequestHandler, 而 TransportRequestHandler 又封装了 master 实际的消息发送 NettyRpcHandler
5. NettyRpcHandler 负责把 nettty 接收到的消息通过持有的 dispatcher 发送到特定 endpoint 的 inbox 消息队列中。
6. 每个 endpoint 配有一个 Inbox，每个 Inbox 在实例化后都会先向对列中添加一个 OnStart 消息。
7. Inbox 的 process 方法完成了一个类的消息到类中具体方法的转发。
8. EndPointRef 是EndPoint 的引用，持有了这个引用才能向 EndPoint 发消息，比如 masterEndPointRef 可以发送消息到 Master.
9. 发送消息到远端如果需要重新创建连接，为了防止阻塞，创建连接的操作是在另外的线程中进行的。
10. 完成 registerRpcEndpoint 后通过 receivers.offer(data) 触发，消耗掉 master inbox 对列中的 OnStart 消息，完成 Master.OnStart方法调用.
11. OnStart 方法中完成 webui restapi metricsSystem 的启动，以及主备选举操作
12. zookeeper 客户端类库使用的是 curator

代码是 spark 2.11，讲真感觉这部分有点混乱，感觉有些地方耦合过大了，像是没有经过认真设计，东一下西一下搭出来的，而且好多类上都没注释，一些关键的方法上注释也不全。

## 具体流程

1. 找到入口点类

    sbin/start-master.sh 脚本中能见到 CLASS="org.apache.spark.deploy.master.Master" 
    
    在启动的时候还能对进程设置优先级，在 sbin/spark-daemon.sh 中，使用 nice 命令。
    
        # nice -n xx + command + args 这就是运行的命令
        execute_command nice -n "$SPARK_NICENESS" "${SPARK_HOME}"/bin/spark-class "$command" "$@"
    

2. 启动时调用 startRpcEnvAndEndpoint 方法，这个方法中会创建一个 RpcEnv 一个 Endpoint 一个 RpcEndpointRef。

        def startRpcEnvAndEndpoint(
            host: String,
            port: Int,
            webUiPort: Int,
            conf: SparkConf): (RpcEnv, Int, Option[Int]) = {
          val securityMgr = new SecurityManager(conf)
          val rpcEnv = RpcEnv.create(SYSTEM_NAME, host, port, conf, securityMgr)
          val masterEndpoint = rpcEnv.setupEndpoint(ENDPOINT_NAME,
            new Master(rpcEnv, rpcEnv.address, webUiPort, securityMgr, conf))
          val portsResponse   masterEndpoint.askWithRetry[BoundPortsResponse](BoundPortsRequest)
          (rpcEnv, portsResponse.webUIPort, portsResponse.restPort)
        }

3. 在 RpcEnv 初始化时会初始化他的一个成员(构造函数中)

        private val transportContext = new TransportContext(transportConf,
            new NettyRpcHandler(dispatcher, this, streamManager))
    
    这里面最关键的就是传入的 NettyRpcHandler 实例，这个就是我们在 netty 中真正处理业务的 handler. 注意 NettyRpcHandler 实例化的时候持有了一个 dispatcher 实例, 后面会具体看 NettyRpcHandler 的业务逻辑，就是在通过这个 dispatcher 实例传递数据，在 netty 底层代码中， rpchandle 被封装到 TransportRequestHandler.

4. 实例化完 RpcEnv 然后就需要启动 netty 服务端了，服务端启动最终创建了一个 TransportServer 的实例，这个类是一个 java 类，在源码的 common/network-common 下面 (org.apache.spark.network.server.TransportServer)
    
    这个类是对 netty 的封装，在这个类中完成了 ServerBootstrap 的创建，参数配置，以及各种 handler 的添加，netty 还属于入门级别，就不具体介绍了。

5. 下面来具体看一看 NettyRpcHandler
    
        private[netty] class NettyRpcHandler(
            dispatcher: Dispatcher,
            nettyEnv: NettyRpcEnv,
            streamManager: StreamManager) extends RpcHandler with Logging {
        
          // 用于保存客户端链接信息
          private val remoteAddresses = new ConcurrentHashMap[RpcAddress, RpcAddress]()
            
          // 有回调函数的 receive 方法
          override def receive(
              client: TransportClient,
              message: ByteBuffer,
              callback: RpcResponseCallback): Unit = {
            val messageToDispatch = internalReceive(client, message)
            // 投递一个 RpcMessage 在后面 inbox 里面匹配有不同的行为。
            dispatcher.postRemoteMessage(messageToDispatch, callback)
          }
          // 没有回调
          override def receive(
              client: TransportClient,
              message: ByteBuffer): Unit = {
            val messageToDispatch = internalReceive(client, message)
            // 投递消息 OneWayMessage 类型的消息到 inbox 里面。
            dispatcher.postOneWayMessage(messageToDispatch)
          }
          // 把消息封装为 case class RequestMessage
          private def internalReceive(client: TransportClient, message: ByteBuffer): RequestMessage = {
            // .. 
          }
        
          override def channelActive(client: TransportClient): Unit = {
            val addr = client.getChannel().remoteAddress().asInstanceOf[InetSocketAddress]
            assert(addr != null)
            val clientAddr = RpcAddress(addr.getHostString, addr.getPort)
            // 告诉每一个 EndpointData 新来了一个连接
            dispatcher.postToAll(RemoteProcessConnected(clientAddr))
          }
        
          override def channelInactive(client: TransportClient): Unit = {
            // ..
          }
        }
    
    在这个 handle 里面完成了封装消息以及把消息送进 dispatcher 的工作

6. dispatcher 消息路由
    
    dispatcher 有一个内部类 EndpointData 这个类中持有特定的 endpoint 实例，并且持有一个 Inbox 实例，这个实例的 process 方法完成了针对特定实例的消息转发,Inbox 实例化的时候还做了另外一件事情，就是把一个 Onstart 消息放到了他自己的消息对列中。这个动作关系着后面我们对 master onStart 方法的调用。
    
          private class EndpointData(
              val name: String,
              val endpoint: RpcEndpoint,
              val ref: NettyRpcEndpointRef) {
            val inbox = new Inbox(ref, endpoint)
          }

          // Inbox 中
          inbox.synchronized {
            messages.add(OnStart)
          }
    
    然后在 dispatcher 中最关键的是下面这个方法，其余的 post 方法都是传递不同的 InboxMessage 子类消息类型到 postMessage

      private def postMessage(
          endpointName: String,
          message: InboxMessage,
          callbackIfStopped: (Exception) => Unit): Unit = {
        val error = synchronized {
          // 获取对应 endpoint 对象，根据名字，endpoints 是一个 hashmap
          val data = endpoints.get(endpointName)
          if (stopped) {
            Some(new RpcEnvStoppedException())
          } else if (data == null) {
            Some(new SparkException(s"Could not find $endpointName."))
          } else {
            // 投递消息到特定的 endpoint 的消息队列
            data.inbox.post(message)
            // receivers 是一个队列，存在这个队列里面的 endpoint 代表有消息要处理
            receivers.offer(data)
            None
          }
        }
        error.foreach(callbackIfStopped)
      }
    
7. 截止到上面的内容都是在 handler 所处的线程中完成的，handle 中完成了到某个具体 endpoint 的分发，但是分发完在哪个处理呢？

    通过在目录里面搜索 inbox.process 关键字我找到了这个类。这个类实现了 Runnable 接口，很显然是要在一个新的线程中跑的嘛..
    
          private class MessageLoop extends Runnable {
            override def run(): Unit = {
              try {
                while (true) {
                  try {
                    // 上面分发的时候我们把一个 endpointData 放到队列 receivers ，然后在这里用到了
                    val data = receivers.take()
                    if (data == PoisonPill) {
                      // PoisonPill 这个玩意应该是代表着结束？
                      receivers.offer(PoisonPill)
                      return
                    }
                    data.inbox.process(Dispatcher.this)
                  } catch {
                    case NonFatal(e) => logError(e.getMessage, e)
                  }
                }
              } catch {
                case ie: InterruptedException => // exit
              }
            }
          }
    
    下面是 MessageLoop 放到线程池中的方法
        
          private val threadpool: ThreadPoolExecutor = {
            val numThreads = nettyEnv.conf.getInt("spark.rpc.netty.dispatcher.numThreads",
              math.max(2, Runtime.getRuntime.availableProcessors()))
            val pool = ThreadUtils.newDaemonFixedThreadPool(numThreads, "dispatcher-event-loop")
            for (i <- 0 until numThreads) {
              pool.execute(new MessageLoop)
            }
            pool
          }
    
    可以通过 spark.rpc.netty.dispatcher.numThread 来控制 dispatcher 线程数。如果后面真正的处理逻辑存在大 io 操作是不是可以通过增加 线程来提高处理效率。当然 master 应该没有什么大 io 操作吧？

8. 下面来看一看 Inbox 的 process 消息派发逻辑
    
          def process(dispatcher: Dispatcher): Unit = {
            var message: InboxMessage = null
            // 取消息的操作是线程安全的，不知道这个 synchronized 对应到 java 里边是什么，回头研究下。
            inbox.synchronized {
              if (!enableConcurrent && numActiveThreads != 0) {
                return
              }
              message = messages.poll()
              if (message != null) {
                numActiveThreads += 1
              } else {
                return
              }
            }
            while (true) {
              // 这个 safeyCall 是为了防止抛出错误。
              safelyCall(endpoint) {
                message match {
                  // 这个是前面通过 postRemoteMessage 投递的消息
                  case RpcMessage(_sender, content, context) =>
                    try {
                      // 这里面就是真正的业务逻辑了，还记得我们我们注册的 endpoint 是什么吗？
                      // 是 master,这里，还有下面几个 case 都是调用 master 的方法
                      endpoint.receiveAndReply(context).applyOrElse[Any, Unit](content, { msg =>
                        throw new SparkException(s"Unsupported message $message from ${_sender}")
                      })
                    } catch {
                      case NonFatal(e) =>
                        context.sendFailure(e)
                        // Throw the exception -- this exception will be caught by the safelyCall function.
                        // The endpoint's onError function will be called.
                        throw e
                    }
                  // 不需要回复的消息
                  case OneWayMessage(_sender, content) =>
                    endpoint.receive.applyOrElse[Any, Unit](content, { msg =>
                      throw new SparkException(s"Unsupported message $message from ${_sender}")
                    })
        
                  case OnStart =>
                    endpoint.onStart()
                    if (!endpoint.isInstanceOf[ThreadSafeRpcEndpoint]) {
                      inbox.synchronized {
                        if (!stopped) {
                          enableConcurrent = true
                        }
                      }
                    }
        
                  case OnStop =>
                    val activeThreads = inbox.synchronized { }
              }
            }
          }

9. 下面先不管 master 到底对消息做了哪些处理，先把注册流程走完。

        val masterEndpoint = rpcEnv.setupEndpoint(ENDPOINT_NAME,
          new Master(rpcEnv, rpcEnv.address, webUiPort, securityMgr, conf))
    
    这是注册 endpoint 并获取 RpcEndpointRef 的过程。最终调用 dispatcher 的 registerRpcEndpoint。
    
          def registerRpcEndpoint(name: String, endpoint: RpcEndpoint): NettyRpcEndpointRef = {
            val addr = RpcEndpointAddress(nettyEnv.address, name)
            val endpointRef = new NettyRpcEndpointRef(nettyEnv.conf, addr, nettyEnv)
            synchronized {
              if (stopped) {
                throw new IllegalStateException("RpcEnv has been stopped")
              }
              if (endpoints.putIfAbsent(name, new EndpointData(name, endpoint, endpointRef)) != null) {
                throw new IllegalArgumentException(s"There is already an RpcEndpoint called $name")
              }
              val data = endpoints.get(name)
              // endpointRefs 这个 map 把前面注册的 endpoint 作为 key ref 作为值注册。
              // 这样假如想在endpoint 的代码中获取 endpointRefs 直接  endpointRefs.get(this) 就可以了.
              endpointRefs.put(data.endpoint, data.ref)
              receivers.offer(data)  // for the OnStart message
            }
            endpointRef
          }

10. 上面的代码中创建了一个 NettyRpcEndpointRef 这个东西的作用是发送消息。比较关键的是下面两个方法。

          override def ask[T: ClassTag](message: Any, timeout: RpcTimeout): Future[T] = {
            nettyEnv.ask(RequestMessage(nettyEnv.address, this, message), timeout)
          }
        
          override def send(message: Any): Unit = {
            require(message != null, "Message is null")
            nettyEnv.send(RequestMessage(nettyEnv.address, this, message))
          }
    
    注意这两个方法构造的 RequestMessage receiver 都是传入的 this,而 send 或者 ask 的具体方法中通过判断 message.receiver.address 是本地的地址 还是远端地址来处理不同的逻辑，如果是本地地址实际上是发送给了 master 这个 endpoint。
    
    这个 NettyRpcEndpointRef 实例是有机会获取到 master 的实例的，但是为什么还用通过向 master 发消息呢？这个后面在介绍 master 具体业务处理流程的时候会有介绍。

11. 前面这些的内容全部都是 master 怎么接收到消息，然后接收到消息怎么处理，但是作为一个老大总不能只会等着小弟们来询问你吧，所以 master 应该也能对外发送消息去控制它的小弟. 那么发送的流程是什么样子呢？
    
    发送是通过 nettyEnv.send 方法完成的，这个方法会判断发送地址是本地还是远端。如果是远端会把消息发送的 outBox 队列中。

          private[netty] def send(message: RequestMessage): Unit = {
            val remoteAddr = message.receiver.address
            if (remoteAddr == address) {
              // Message to a local RPC endpoint.
              try {
                dispatcher.postOneWayMessage(message)
              } catch {
                case e: RpcEnvStoppedException => logWarning(e.getMessage)
              }
            } else {
              // Message to a remote RPC endpoint.
              postToOutbox(message.receiver, OneWayOutboxMessage(serialize(message)))
            }
          }
    
    postToOutbox 函数首先判断 client 连接还存在不存在， 如果存在则直接发送消息，否则判断以前有没有创建过连接到给定地 outbox，如果没有就新建一个，如果已经创建了就返回原来的。然后再通过 outBox 的 send() 方法把消息送进对应 outBox 队列。
    
      private def postToOutbox(receiver: NettyRpcEndpointRef, message: OutboxMessage): Unit = {
        if (receiver.client != null) {
          // 这里的 client 是  org.apache.spark.network.client.TransportClient
          // 这个东西是 java 写的在 common/network-common 包下面
          // 实际上是对底层 netty 的封装
          message.sendWith(receiver.client)
        } else {
          require(receiver.address != null,
            "Cannot send message to client endpoint with no listen address.")
          // 获取 outbox
          val targetOutbox = {
              val outbox = outboxes.get(receiver.address)
            if (outbox == null) {
              val newOutbox = new Outbox(this, receiver.address)
              val oldOutbox = outboxes.putIfAbsent(receiver.address, newOutbox)
              if (oldOutbox == null) {
                newOutbox
              } else {
                oldOutbox
              }
            } else {
              outbox
            }
          }
          if (stopped.get) {
            // It's possible that we put `targetOutbox` after stopping. So we need to clean it.
            outboxes.remove(receiver.address)
            targetOutbox.stop()
          } else {
            // outBox 的 send 方法。
            targetOutbox.send(message)
          }
        }
      }
    
12. 这里可以和 inbox 做一下对比，inbox 是有一个专门的线程池负责处理接收到的消息，但是 outbox 在他的消息的产生线程中好像就可以直接发送，那么这里为什么还要有与入队列相关的操作呢？

    想一下，如果当前并没有与远程节点建立连接，那么发送消息前的三次握手操作其实会导致较长一段时间的线程阻塞。所以这里做的工作是把与远程节点建立连接的操作放到一个单独的线程中。同时，如果在连接建立的过程中，如果有多个发送请求会造成消息的堆积，所以在连接建立后线程并不会马上结束，而是接着处理消息队列中的消息，直到队列为空。下面的代码就完成了上面的逻辑。
    
          def send(message: OutboxMessage): Unit = {
            val dropped = synchronized {
              if (stopped) {
                true
              } else {
                messages.add(message)
                false
              }
            }
            if (dropped) {
              message.onFailure(new SparkException("Message is dropped because Outbox is stopped"))
            } else {
              // 这个方法,内完成了上面所说的逻辑
              drainOutbox()
            }
          }
          
          // 重点
          private def drainOutbox(): Unit = {
            var message: OutboxMessage = null
            synchronized {
              if (stopped) {
                return
              }
              // future 接口不为空, 说明正在建立连接
              // 不用重新启动线程池建立连接了，直接返回，
              if (connectFuture != null) {
                // We are connecting to the remote address, so just exit
                return
              }
              // client 为空，说明没有与远程节点的连接，需要建立
              if (client == null) {
                // 这个方法中通过线程池来完成连接创建
                launchConnectTask()
                return
              }
              if (draining) {
                // There is some thread draining, so just exit
                return
              }
              message = messages.poll()
              if (message == null) {
                return
              }
              draining = true
            }
            // 消费消息
            while (true) {
              try {
                val _client = synchronized { client }
                if (_client != null) {
                  message.sendWith(_client)
                } else {
                  assert(stopped == true)
                }
              } catch {
                case NonFatal(e) =>
                  handleNetworkFailure(e)
                  return
              }
              synchronized {
                if (stopped) {
                  return
                }
                message = messages.poll()
                if (message == null) {
                  draining = false
                  return
                }
              }
            }
          }
    
    下面这个方法是具体建立连接的方法：
    
          private def launchConnectTask(): Unit = {
            connectFuture = nettyEnv.clientConnectionExecutor.submit(new Callable[Unit] {
        
              override def call(): Unit = {
                try {
                  // 这个方法再继续往下跟就到了 netty 封装的底层代码了。
                  val _client = nettyEnv.createClient(address)
                  outbox.synchronized {
                    client = _client
                    if (stopped) {
                      closeClient()
                    }
                  }
                } catch {
                  case ie: InterruptedException =>
                    // exit
                    return
                  case NonFatal(e) =>
                    outbox.synchronized { connectFuture = null }
                    handleNetworkFailure(e)
                    return
                }
                outbox.synchronized { connectFuture = null }
                
                // 完成线程创建后再消费掉堆积的消息
                drainOutbox()
              }
            })
          }

13. 内容虽然略多，但是其核心就是在做三个事情: 1. 启动 netty 服务器用于监听网络请求 2. 注册 endpoint 用于处理具体请求 3. 注册 endpointRef 用于通信.

14. 完成上面的工作，其实基本上就 master 就已经启动成功了，但是我们还需要调用 onStart 方法来完成一些小东西的启动工作，或者是 leader 选举之类的工作，还记得在 registerRpcEndpoint 的那一步有一个 receivers.offer(data) 操作。这个操作的目的就是为了完成 OnStart 方法的触发。这里的调用时机也很巧妙，因为在 onStart 方法中可能需要用到 endpointRef 对象，所以这里在 endpointRef 注册完之后才可能触发 Onstart.

    还记得我们前面提过的 inbox massage 队列中有一个 OnStart 消息是在 Inbox 构造函数中添加进去的，receivers.offer(data) 配合着 OnStart 消息完成了对 OnStart 方法的调用。(match 到 inbox.process 的 Onstart 项)
    
15. 然后在 OnStart 方法中所做的事情有: 
    
          override def onStart(): Unit = {
            // 启动 webui
            webUi = new MasterWebUI(this, webUiPort)
            webUi.bind()
            masterWebUiUrl = "http://" + masterPublicAddress + ":" + webUi.boundPort
      
            // 添加一个触发检测 worker 超时的任务
            checkForWorkerTimeOutTask = forwardMessageThread.scheduleAtFixedRate(new Runnable {
              override def run(): Unit = Utils.tryLogNonFatalError {
                self.send(CheckForWorkerTimeOut)
              }
            }, 0, WORKER_TIMEOUT_MS, TimeUnit.MILLISECONDS)
            // restful 形式的 api 接口服务
            if (restServerEnabled) {
              val port = conf.getInt("spark.master.rest.port", 6066)
              restServer = Some(new StandaloneRestServer(address.host, port, conf, self, masterUrl))
            }
            restServerBoundPort = restServer.map(_.start())
            
            // $SPARK_HOME/conf/metrics.properties 是相关配置文件
            // 系统信息采集
            masterMetricsSystem.registerSource(masterSource)
            masterMetricsSystem.start()
            applicationMetricsSystem.start()
            masterMetricsSystem.getServletHandlers.foreach(webUi.attachHandler)
            applicationMetricsSystem.getServletHandlers.foreach(webUi.attachHandler)
            // 别管这个还在开发
            val serializer = new JavaSerializer(conf)
            // 根据集群模式进行选举
            val (persistenceEngine_, leaderElectionAgent_) = RECOVERY_MODE match {
              case "ZOOKEEPER" =>
                logInfo("Persisting recovery state to ZooKeeper")
                
                // ZK 使用 org.apache.curator.framework.recipes.leader.LeaderLatch
                // 要比原生的 zk 库好用
                val zkFactory =
                  new ZooKeeperRecoveryModeFactory(conf, serializer)
                (zkFactory.createPersistenceEngine(), zkFactory.createLeaderElectionAgent(this))
              // ...
            }
            persistenceEngine = persistenceEngine_
            leaderElectionAgent = leaderElectionAgent_
          }

16. Spark Master 的整个启动流程大概就是这个样子了。
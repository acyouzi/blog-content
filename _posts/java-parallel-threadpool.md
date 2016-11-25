---
title: 线程池 ThreadPoolExecutor 的使用和实现分析
date: 2016-11-25 17:57:38
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- java
	- 线程池
---

### ThreadPool 使用
1. 创建线程池，可以通过 new ThreadPoolExecutor 来创建， ThreadPoolExecutor 中可以设置如下一些参数：
   
        public ThreadPoolExecutor(int corePoolSize,
                                    int maximumPoolSize,
                                    long keepAliveTime,
                                    TimeUnit unit,
                                    BlockingQueue<Runnable> workQueue,
                                    ThreadFactory threadFactory,
                                    RejectedExecutionHandler handler)

    * corePoolSize 线程池中线程的数量
    * maximumPoolSize 线程最大数量，当任务过多， corePoolSize 指定的线程无法满足需求导致 workQueue 被任务塞满时会扩大线程池中线程的数量，但不会超出 maximumPoolSize 设置的数量。
    * keepAliveTime 指定超过 corePoolSize 部分的空闲线程，在多长空闲时间后被销毁
    * unit keepAliveTime 的时间单位
    * workQueue 被提交但是还没执行的任务会放到这个队列中，可以列举几种队列：

            SynchronousQueue 每一个插入操作都要等待对应的删除操作，不会在队列中真正保存任务。
            ArrayBlockingQueue 有界队列
            LinkedBlockingQueue 无界队列
            PriorityBlockingQueue 带有优先级的任务队列

    * threadFactory 线程工厂用于创建线程
    * handler 拒绝策略，任务太多时来不及处理时使用。

实际上这些参数我们一般不用自己去写， Executors 类已经为我们做了封装了，Executors 中有如下一些方法。

        // 创建固定大小的线程池，使用 LinkedBlockingQueue
        public static ExecutorService newFixedThreadPool(int nThreads) 

        // 只有一个线程的线程池
        public static ExecutorService newSingleThreadExecutor()

        // corePoolSize = 0, maximumPoolSize = Integer.MAX_VALUE, keepAliveTime = 60s , workQueue = SynchronousQueue
        public static ExecutorService newCachedThreadPool() 

        // ScheduledExecutorService 是 ThreadPoolExecutor 的子类，可以设置任务执行时间，周期，只有一个线程
        public static ScheduledExecutorService newSingleThreadScheduledExecutor()

        // 同上，可以指定多个线程
        public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize)

2. 调用 execute 或者 submit 提交任务到线程池
3. 对于 ScheduledExecutorService 类型需要调用如下几个方法

        // 给定延时之后调用一次任务
        public ScheduledFuture<?> schedule(Runnable command,long delay, TimeUnit unit);
        
        // 每隔固定的时间调用一次任务，周期性的
        // 任务不会出现堆叠，假设设定的间隔时间是 2s ，但是实际任务执行了 8s,
        //  那么任务实际上就变成 8s 调度一次了
        public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                    long initialDelay,
                                                    long period,
                                                    TimeUnit unit);

        // 任务执行完后延时给定时间再进行下次调用
        public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                        long initialDelay,
                                                        long delay,
                                                        TimeUnit unit);

### 实现
线程池的实现，原理上很容易理解： 开启多个线程，线程会循环的从一个对列中读取提交的任务，如果队列空线程就阻塞起来。不为空就读取任务，然后执行。

1. 下面是几个与业务逻辑有关的常量，为了下面代码能看明白，需要先明确一下：

        COUNT_BITS = Integer.SIZE - 3;   // 29
        CAPACITY   = (1 << COUNT_BITS) - 1;  // 高 3 位为0，低 29 位为 1
        RUNNING    = -1 << COUNT_BITS;  // 111 + 29 个 0 ，
                                        // 这几个状态中只有 RUNNING 最高位为 1 ,也就是只有 RUNNING 小于 0
        SHUTDOWN   =  0 << COUNT_BITS;  // 32 个 0
        STOP       =  1 << COUNT_BITS;  // 001 + 29 个 0
        TIDYING    =  2 << COUNT_BITS;  // 010 + 29 个 0
        TERMINATED =  3 << COUNT_BITS;  // 011 + 29 个 0

        // ctl 保存的是一个 32 位的 int 类型数
        // 高 3 位用于存储状态信息，低 29 位用于存储线程数量
        ctl = new AtomicInteger(ctlOf(RUNNING, 0));

2. 调用 execute

        public void execute(Runnable command) {
            if (command == null)
                throw new NullPointerException();
            int c = ctl.get();
            if (workerCountOf(c) < corePoolSize) {
                if (addWorker(command, true))
                    return;
                c = ctl.get();
            }
            if (isRunning(c) && workQueue.offer(command)) {
                int recheck = ctl.get();
                if (! isRunning(recheck) && remove(command))
                    reject(command);
                else if (workerCountOf(recheck) == 0)
                    addWorker(null, false);
            }
            else if (!addWorker(command, false))
                reject(command);
        }

    首先判断线程数量有没有超过 corePoolSize, 如果没有直接调用 addWorker 创建新的 worker

    如果线程数已经超过了 corePoolSize, 则调用 workQueue.offer 把任务加入队列中，然后在检查一遍线程池状态，保证其正常运行

    如果线程池状态不再是 RUNNING 或者 workQueue.offer 添加任务失败(可能是因为队列满了), 则调用 addWorker 创建新的 worker。

3. addWorker ， 上面部分三次调用 addWorker 分别有不同的参数组合

    * core == true , 代表线程池中线程数还没达到 corePoolSize 值，新建一个任务不能超过 corePoolSize 的值
    * core == false , 代表以 maximumPoolSize 的值作为线程池大小的判断标准
    * firstTask 创建 worker 之后执行的第一个任务，可以为空。

            private boolean addWorker(Runnable firstTask, boolean core) {
                // cas 增加 worker 的计数器值
                retry:
                for (;;) {
                    int c = ctl.get();
                    int rs = runStateOf(c);

                    if (rs >= SHUTDOWN &&
                        ! (rs == SHUTDOWN &&
                        firstTask == null &&
                        ! workQueue.isEmpty()))
                        return false;

                    for (;;) {
                        int wc = workerCountOf(c);
                        if (wc >= CAPACITY ||
                            wc >= (core ? corePoolSize : maximumPoolSize))
                            return false;
                        if (compareAndIncrementWorkerCount(c))
                            break retry;
                        c = ctl.get();  // Re-read ctl
                        if (runStateOf(c) != rs)
                            continue retry;
                        // else CAS failed due to workerCount change; retry inner loop
                    }
                }

                boolean workerStarted = false;
                boolean workerAdded = false;
                Worker w = null;
                try {
                    w = new Worker(firstTask);
                    final Thread t = w.thread;
                    if (t != null) {
                        final ReentrantLock mainLock = this.mainLock;
                        mainLock.lock();
                        try {
                            // Recheck while holding lock.
                            // Back out on ThreadFactory failure or if
                            // shut down before lock acquired.
                            int rs = runStateOf(ctl.get());

                            if (rs < SHUTDOWN ||
                                (rs == SHUTDOWN && firstTask == null)) {
                                if (t.isAlive()) // precheck that t is startable
                                    throw new IllegalThreadStateException();
                                workers.add(w);
                                int s = workers.size();
                                if (s > largestPoolSize)
                                    largestPoolSize = s;
                                workerAdded = true;
                            }
                        } finally {
                            mainLock.unlock();
                        }
                        if (workerAdded) {
                            t.start();
                            workerStarted = true;
                        }
                    }
                } finally {
                    if (! workerStarted)
                        addWorkerFailed(w);
                }
                return workerStarted;
            }

    添加任务完成调用 t.start 启动线程，就可以看 worker 的 run 方法了。

3. worker 的 run 方法直接调用 runWorker 方法 
    在 runWorker 方法中，循环从 workQueue 中获取任务执行，如果设置了 firstTask 那么第一个任务先执行 firstTask。
    然后在执行我们任务的 run 方法之前会先调用 beforeExecute(wt, task)，任务执行完之后会调用 afterExecute 方法。在使用线程池时我们可以重写这两个方法。

        final void runWorker(Worker w) {
            Thread wt = Thread.currentThread();
            Runnable task = w.firstTask;
            w.firstTask = null;
            w.unlock(); // allow interrupts
            boolean completedAbruptly = true;
            try {
                while (task != null || (task = getTask()) != null) {
                    w.lock();
                    if ((runStateAtLeast(ctl.get(), STOP) ||
                        (Thread.interrupted() &&
                        runStateAtLeast(ctl.get(), STOP))) &&
                        !wt.isInterrupted())
                        wt.interrupt();
                    try {
                        beforeExecute(wt, task);
                        Throwable thrown = null;
                        try {
                            task.run();
                        } catch (RuntimeException x) {
                            thrown = x; throw x;
                        } catch (Error x) {
                            thrown = x; throw x;
                        } catch (Throwable x) {
                            thrown = x; throw new Error(x);
                        } finally {
                            afterExecute(task, thrown);
                        }
                    } finally {
                        task = null;
                        w.completedTasks++;
                        w.unlock();
                    }
                }
                completedAbruptly = false;
            } finally {
                processWorkerExit(w, completedAbruptly);
            }
        }

4. 在循环从队列获取任务的过程中我们会多次调用 getTask 方法，但我们设置允许超时退出，或者线程池中线程数大于 corePoolSize 时，会调用 workQueue.poll 方法，这个方法会等待一定时间，不会一直阻塞，如果等待时间内没有等到任务会返回 null. 否则调用 workQueue.take()，这个方法会一直阻塞，直到拿到任务。

        private Runnable getTask() {
            boolean timedOut = false; // Did the last poll() time out?

            for (;;) {
                int c = ctl.get();
                int rs = runStateOf(c);

                // Check if queue empty only if necessary.
                if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                    decrementWorkerCount();
                    return null;
                }

                int wc = workerCountOf(c);

                // Are workers subject to culling?
                boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

                if ((wc > maximumPoolSize || (timed && timedOut))
                    && (wc > 1 || workQueue.isEmpty())) {
                    if (compareAndDecrementWorkerCount(c))
                        return null;
                    continue;
                }

                try {
                    Runnable r = timed ?
                        workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                        workQueue.take();
                    if (r != null)
                        return r;
                    timedOut = true;
                } catch (InterruptedException retry) {
                    timedOut = false;
                }
            }
        }

5. 如果 getTask 返回了 null 就会跳出 while 循环 调用 processWorkerExit(w, completedAbruptly) 方法在退出线程前做一些处理，或者退出或者，从新创建一个 Worker.

6. 另一个需要了解的方法是 shutdown() 
    shutdown 的主要作用是设置线程池的运行状态为 SHUTDOWN, 而非强制关闭。terminated() 在线程关闭时会被调用，可以重写这个方法监控线程关闭的情况。

### submit 与 execute 的区别
submit 会把任务封装到一个 FutureTask 中，并且 submit 会返回一个 FutureTask 实例，这样我们就可以用这个实例去查询，获取任务的一些状态。

### ScheduledThreadPoolExecutor 的实现

ScheduledThreadPoolExecutor 有一个内部类，DelayedWorkQueue 使用数组存放队列元素，在这个队列的 take 和 poll 方法实现中，会根据我们设置的延时调用 available.awaitNanos(delay) 等待一定时间再返回。

    public RunnableScheduledFuture<?> take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                RunnableScheduledFuture<?> first = queue[0];
                if (first == null)
                    available.await();
                else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        return finishPoll(first);
                    first = null; // don't retain ref while waiting
                    if (leader != null)
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && queue[0] != null)
                available.signal();
            lock.unlock();
        }
    }

另外，我们提交到 worker 执行的实际上是一个继承自 FutureTask 的 ScheduledFutureTask 类的 run 方法，这个 run 方法中会对定时任务和周期调度任务分别做不同的处理。

    public void run() {
        // period != 0 判读有没有设置调度延时
        boolean periodic = isPeriodic();
        if (!canRunInCurrentRunState(periodic))
            cancel(false);
        else if (!periodic)
            // 没有设置周期调度延时(用 schedule 提交)直接调用父类，也就是 FutureTask 的 run 方法
            ScheduledFutureTask.super.run();
        // 否则(用 scheduleAtFixedRate scheduleWithFixedDelay 提交)调用父类 runAndReset 方法 
        // 这个方法调用完会再设置 future 到 initial state
        else if (ScheduledFutureTask.super.runAndReset()) {
            // 计算下一次调度时间,设置成员变量 period
            setNextRunTime();
            // 再次把 task 加入到 workQueue 中
            reExecutePeriodic(outerTask);
        }
    }


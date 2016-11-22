---
title: ReentrantLock 重入锁使用与实现
date: 2016-11-20 23:12:03
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- java
	- 并发
---

### ReentrantLock 重入锁
1. 重入锁的名字是因为在同一个线程内他可以反复加锁，一个线程可以同时获得多把锁。因而可以写出如下代码:
    
        lock.lock();
        lock.lock();
        try {
            // do something 
        }finally {
            lock.unlock();
            lock.unlock();
        }

2. 使用 lock.lockInterruptibly() 重入锁可以对在等待锁期间对中断做出相应，示例如下：

        try {
            lock.lockInterruptibly();
            // do something
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            if ( lock.isHeldByCurrentThread() ) {
                lock.unlock();
            }
        }

3. 使用 tryLock() 可以进行有限时间的等待：

        try {
            if (lock.tryLock(1000, TimeUnit.MICROSECONDS)) {
                // do something
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    tryLock 第一个参数是时间，第二个参数是时间的单位。也可以不带参数，进行一次锁请求，失败立即返回。

4. 公平锁与非公平锁：ReentrantLock 默认是非公平锁，也就是在唤醒时随便从等待队列中挑选一个线程唤醒，不能保证先来先得。可以通过使用构造函数 public ReentrantLock(boolean fair) 来创建公平锁。公平锁的性能相对要低一点。

### 重入锁实现
先来看几种状态

* SIGNAL -1 : 表示节点的后继节点已经阻塞或即将阻塞，需要信号唤醒
* CANCELLED 1 : 因为超时或中断取消等待 
* CONDITION -2 : 等待某个条件而被阻塞，在条件队列里面
* PROPAGATE -3 : 共享模式中使用
* 0 : None of the above

以如下代码为例：

    public static ReentrantLock lock = new ReentrantLock();
    public static int i;
    public static Runnable run = () -> {
        for (int j = 0; j < 10; j++) {
            lock.lock();
            i++;
            lock.unlock();
        }
    };
    public static void main(String[] args) {
        List<Thread> list = Stream.generate(()-> new Thread(run)).limit(3).collect(Collectors.toList());
        list.stream().forEach(t -> t.start());
        list.stream().forEach(t -> {
            try {
                t.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        System.out.println(i);
    }

1. 首先创建的是一个非公平锁，所以 ReentrantLock 的 sync 持有的是 NonfairSync 实例，当第一个线程去执行 lock 方法获取锁时：

        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

    cas 操作成功获得锁，设置 ExclusiveOwnerThread 继续执行业务逻辑

2. 第二个线程执行 lock 方法时，假设第一个线程还没有释放锁，这时 cas 操作失败，执行 acquire(1) , 在 acquire 中：

        public final void acquire(int arg) {
            if (!tryAcquire(arg) &&
                acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
                selfInterrupt();
        }

    首先调用 tryAcquire 注意这里调用的是 NonfairSync 的 tryAcquire 方法，最终调用 nonfairTryAcquire 再次尝试获取锁，或者可能是已经持有锁的线程再次获取锁把 state + 1

        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

    这里返回 false 然后 acquire 中调用 addWaiter 把节点加入队列尾部：

        private Node addWaiter(Node mode) {
            Node node = new Node(Thread.currentThread(), mode);
            // Try the fast path of enq; backup to full enq on failure
            Node pred = tail;
            if (pred != null) {
                node.prev = pred;
                if (compareAndSetTail(pred, node)) {
                    pred.next = node;
                    return node;
                }
            }
            enq(node);
            return node;
        }

    这里队列为空时或者 cas 操作失败时会执行 enq(node) 操作：

        private Node enq(final Node node) {
            for (;;) {
                Node t = tail;
                if (t == null) { // Must initialize
                    if (compareAndSetHead(new Node()))
                        tail = head;
                } else {
                    node.prev = t;
                    if (compareAndSetTail(t, node)) {
                        t.next = node;
                        return t;
                    }
                }
            }
        }

    入队列完成后执行 acquireQueued 方法：

        final boolean acquireQueued(final Node node, int arg) {
            boolean failed = true;
            try {
                boolean interrupted = false;
                for (;;) {
                    final Node p = node.predecessor();
                    if (p == head && tryAcquire(arg)) {
                        setHead(node);
                        p.next = null; // help GC
                        failed = false;
                        return interrupted;
                    }
                    if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                        interrupted = true;
                }
            } finally {
                if (failed)
                    cancelAcquire(node);
            }
        }

    在这个方法中判断节点的前驱结点是不是头结点，如果是头结点，再次尝试获取锁，如果不是调用 shouldParkAfterFailedAcquire 判断是不是应该挂起：

        private static boolean shouldParkAfterFailedAcquire(Node pre, Node node) {
            int ws = pred.waitStatus;
            if (ws == Node.SIGNAL)
                return true;
            if (ws > 0) {
                do {
                    node.prev = pred = pred.prev;
                } while (pred.waitStatus > 0);
                pred.next = node;
            } else {
                compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
            }
            return false;
        }

    如果前一个节点也是 Node.SIGNAL 状态，那就返回 true ,否则判断前一个节点的状态是不是被取消的状态，如果是移除节点。否则改变前一个节点的状态为 Node.SIGNAL 。最后返回false。

    这时再次跳到 acquireQueued 的 for 循环中，然后再次进入 shouldParkAfterFailedAcquire，如果这次返回结果为 true 则调用 parkAndCheckInterrupt 方法，这个方法会阻塞当前线程。

        private final boolean parkAndCheckInterrupt() {
            LockSupport.park(this);
            return Thread.interrupted();
        }

    这里的阻塞需要在 unlock 中唤醒才能够继续执行，如果线程没有被中断过，则返回fales 返回到 acquireQueued 中去继续锁的竞争。

3. 公平锁与非公平锁的区别在于 tryAcquire 方法，公平锁的 tryAcquire 方法 ： 

        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

    主要在这一句 if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) 保证等待时间最长的线程才能参与锁竞争。

    另外一个区别在 locl 方法，不会再去尝试获取锁，而是直接调用 acquire 方法。
    
        final void lock() {
            acquire(1);
        }
    

4. unlock 会调用 release 方法， 

        public final boolean release(int arg) {
            if (tryRelease(arg)) {
                Node h = head;
                if (h != null && h.waitStatus != 0)
                    unparkSuccessor(h);
                return true;
            }
            return false;
        }

    其中 tryRelease 方法在一个线程持有的锁全部释放时放回 true

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

    当 tryRelease 返回 true 会，会调用 unparkSuccessor 唤醒等待的线程。

        private void unparkSuccessor(Node node) {
            int ws = node.waitStatus;
            // 清理状态
            if (ws < 0)
                compareAndSetWaitStatus(node, ws, 0);
            Node s = node.next;
            if (s == null || s.waitStatus > 0) {
                s = null;
                for (Node t = tail; t != null && t != node; t = t.prev)
                    if (t.waitStatus <= 0)
                        s = t;
            }
            if (s != null)
                LockSupport.unpark(s.thread);
        }
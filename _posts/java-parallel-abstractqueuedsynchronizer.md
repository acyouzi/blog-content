---
title: AbstractQueuedSynchronizer 的共享锁独占锁实现
date: 2016-11-21 15:42:39
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- java
	- 并发
---

AbstractQueuedSynchronizer 可用作为一个同步工具的基础，持有一个volatile int state的属性，表示同步的状态。使用 cas 操作修改 state 保证线程安全。内部使用 LockSupport 来阻塞唤醒线程，维护链表来表示等待锁的队列。

### 链表的节点结构如下

* int waitStatus 有如下四中状态 
    SIGNAL -1 : 表示节点的后继节点已经阻塞或即将阻塞，需要信号唤醒
    CANCELLED 1 : 因为超时或中断取消等待 
    CONDITION -2 : 等待某个条件而被阻塞，在条件队列里面
    PROPAGATE -3 : 共享模式中使用，表示当前场景下后续的acquireShared能够得以执行
    0 : None of the above
* Node prev	前驱节点
* Node next	后继节点。
* Node nextWaiter	存储condition队列中的后继节点
* Thread thread	关联的线程实例


### 独占锁
需要重写如下两个方法

    protected boolean tryAcquire(int arg)
    protected boolean tryRelease(int arg)

####  获取锁
独占模式在同一时刻只能有一个线程获得锁，调用 acquire 获取锁：

    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

这里面首先会调用我们重写的 tryAcquire 方法来尝试获取锁，如果获取失败加入队列，然后调用 acquireQueued 。

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

在这里面会再次尝试获取锁，即使被阻塞，唤醒之后还会进入这里面尝试获取锁。当能够执行到 parkAndCheckInterrupt 时会被阻塞：

    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }

当被阻塞的线程被唤醒时会执行 Thread.interrupted() 判断线程有没有被中断过，同时会清除中断状态，返回 true or false 。从这里开这种方法对中断好像不太敏感，要等到被唤醒并且抢到锁时才能对中断做出响应。

#### 释放锁

    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

首先调用我们自己定义的释放锁的方法，如果释放成功，从等待队列中选一个线程唤醒 LockSupport.unpark(s.thread);

#### 关于 tryAcquire tryRelease 的重写，以及公平锁
参展一些现有的实现 tryAcquire tryRelease 中一般需要使用 cas 修改 state 值，然后修改独占模式的运行线程引用。

    protected boolean tryAcquire(int arg) {
        if (compareAndSetState(0,1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }
    protected boolean tryRelease(int arg) {
        if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
        if (compareAndSetState(1,0)) {
            setExclusiveOwnerThread(null);
            return true;
        }
        return false;
    }

如果要实现公平锁，一般在 tryAcquire 的 if 语句中加一个 判断当前节点是不是等待最久的节点的函数，让对列中等待最久的节点获得锁，同时禁止不在队列里面的节点获取锁。

#### tryAcquireNanos
用于实现有限时间的锁等待

关键代码是调用的 doAcquireNanos 函数，关键部分如下：

    for (;;) {
        final Node p = node.predecessor();
        if (p == head && tryAcquire(arg)) {
            setHead(node);
            p.next = null; // help GC
            failed = false;
            return true;
        }
        nanosTimeout = deadline - System.nanoTime();
        if (nanosTimeout <= 0L)
            return false;
        if (shouldParkAfterFailedAcquire(p, node) &&
            nanosTimeout > spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanosTimeout);
        if (Thread.interrupted())
            throw new InterruptedException();
    }

使用 LockSupport.parkNanos(this, nanosTimeout) 实现有限时间挂起。 


#### acquireInterruptibly 
可中断的锁等待

关键代码是如下部分：
    
    for (;;) {
        final Node p = node.predecessor();
        if (p == head && tryAcquire(arg)) {
            setHead(node);
            p.next = null; // help GC
            failed = false;
            return;
        }
        if (shouldParkAfterFailedAcquire(p, node) &&
            parkAndCheckInterrupt())
            throw new InterruptedException();
    }

当 parkAndCheckInterrupt 挂起结束返回，参与锁的竞争时，如果被设置过中断则直接抛出中断异常，不是等到获取锁然后在返回时中断。

### 共享锁
需要重写如下两个方法：

    protected int tryAcquireShared(int arg)
    protected boolean tryReleaseShared(int arg)

// 未完

### protected boolean isHeldExclusively()
这个函数也需要自己实现，不同的业务逻辑有不同的实现方式，如重入锁中有如下实现：

    protected final boolean isHeldExclusively() {
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

在 ThreadPoolExecutor 中有不同的实现

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }



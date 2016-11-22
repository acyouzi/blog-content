---
title: Condition 的使用与原理
date: 2016-11-21 17:12:40
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- java
	- 并发
---

Condition 与重入锁相关联，其关系类似于 synchronized 与 wait notify 之间的关系。

### Condition 的方法
* public final void await() 使当前线程等待同时释放锁 
* public final long awaitNanos(long nanosTimeout) 有限时间的等待
* public final boolean awaitUntil(Date deadline) 等待，直到某个时间之前
* public final boolean await(long time, TimeUnit unit) 有限时间等待
* public final void awaitUninterruptibly() 等待，并且不响应中断
* public final void signal()
* public final void signalAll()

### 使用示例

    public static ReentrantLock lock = new ReentrantLock();
    public static Condition condition = lock.newCondition();
    public static Runnable run = () -> {
        try {
            lock.lock();
            System.out.println(Thread.currentThread().getName());
            condition.await();
            System.out.println(Thread.currentThread().getName());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    };
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(run);
        t.start();
        Thread.sleep(8000);
        lock.lock();
        condition.signal();
        lock.unlock();
        t.join();
    }

类似于 wait/notify ，condition.await() condition.notify() 的调用必须在lock.lock()和lock.unlock() 之间。

### 实现
Condition 由 AbstractQueuedSynchronizer 的内部类 ConditionObject 实现。

获取 ConditionObject 需要调用 ReentrantLock 实现的 newCondition() 方法：

    final ConditionObject newCondition() {
        return new ConditionObject();
    }

#### await 
1. 在 await 方法中，首先调用 addConditionWaiter 清理 condition queue 中不是CONDITION的状态，然后把节点加入到 condition queue 尾部。

        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

2. 然后在 fullyRelease 方法中会尝试去释放线程持有的锁 , fullyRelease 方法中调用 release 函数，在 release 方法中调用 tryRelease ,重入锁在 tryRelease 中判断释放锁的线程是不是持有锁的线程，如果不是抛出 IllegalMonitorStateException 异常。

        final int fullyRelease(Node node) {
            boolean failed = true;
            try {
                int savedState = getState();
                if (release(savedState)) {
                    failed = false;
                    return savedState;
                } else {
                    throw new IllegalMonitorStateException();
                }
            } finally {
                if (failed)
                    node.waitStatus = Node.CANCELLED;
            }
        }

3. 在 while 循环中，调用 isOnSyncQueue 判断当前节点在不在 sync queue 中等待再次获取锁，如果不在 sync queue 并且没有发生中断事件，会一直循环下去。在循环中会挂起线程，挂起结束后调用 checkInterruptWhileWaiting 函数，这个函数内部会判断如果发生过中断会调用 transferAfterCancelledWait 函数。

        final boolean transferAfterCancelledWait(Node node) {
            if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
                enq(node);
                return true;
            }
            while (!isOnSyncQueue(node))
                Thread.yield();
            return false;
        }

    transferAfterCancelledWait 方法中会去尝试修改节点的状态，如果失败会一直等待(Thread.yield())，直到这个节点被加入到 sync queue 中才会返回继续执行。 

4. 当从 while 循环中退出后会参与到锁竞争中，最后处理中断信息。

#### signal
从下面的源码中可以看到下面的 signal 过程并没有去释放锁，主要操作仅仅是把等待的线程从 condition queue 换到 sync queue 中。

1. 首先判断释放锁的线程有没有持有锁，如果没有持有锁，抛出异常。

        public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }

2. 然后调用 doSignal 最终会调到 transferForSignal, 在这个方法中会把要唤醒的节点加入到 sync queue 中, unpark 对应线程。

        final boolean transferForSignal(Node node) {
            if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
                return false;
            Node p = enq(node);
            int ws = p.waitStatus;
            //状态为cancel或者waitStatus修改失败则唤醒。
            if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
                LockSupport.unpark(node.thread);
            return true;
        }

    按照常理来讲这个 if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL)) 判断应该很少为真才对，唤醒应该是等到 unlock 的时候再做吧。

#### awaitUninterruptibly 
这个方法主要的不同是在 while 循环中并不会因为中断问题而跳槽循环，而是直到 isOnSyncQueue 这个条件满足才退出参与锁竞争、响应中断。

    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if (Thread.interrupted())
            interrupted = true;
    }

#### awaitNanos
这个方法与 await 的区别是挂起使用的是 LockSupport.parkNanos(this, nanosTimeout) 方法。




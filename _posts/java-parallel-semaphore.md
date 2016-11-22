---
title: 信号量的原理及应用
date: 2016-11-22 19:22:14
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- java
	- 并发
---

### 信号量方法介绍
* public Semaphore(int permits) 创建使用非公平策略的信号量
* public Semaphore(int permits, boolean fair) 指定信号量是否使用非公平策略
* public void acquire() 请求，可中断，有重载方法可以请求多个
* public void acquireUninterruptibly() 不可中断
* public boolean tryAcquire() 尝试请求，失败返回false 
* public boolean tryAcquire(long timeout, TimeUnit unit) 指定时间的尝试请求 
* public boolean tryAcquire(int permits) 指定个数的尝试请求
* public boolean tryAcquire(int permits, long timeout, TimeUnit unit) 指定个数和等待时间的尝试请求。
* public void release(int permits) 释放。
* public void release()
* public int drainPermits() 请求目前全部可用的信号，返回信号个数。

### 实现

1. 创建信号量实例，指定初始个数
    Semaphore semp = new Semaphore(3) 最终通过 setState(permits) 设置 state 变量的初始值。不是信号量的上限或者下限数目，仅仅是一个初始值。

2. 请求信号量
    semp.acquire() 最终会调用 AbstractQueuedSynchronizer 的 acquireSharedInterruptibly 方法 

        public final void acquireSharedInterruptibly(int arg)
                throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            if (tryAcquireShared(arg) < 0)
                doAcquireSharedInterruptibly(arg);
        }

    这个方法中会调用 tryAcquireShared 与 重入锁一样，公平与非公平信号量的区别体现在这里，如果是公平的信号量会保证参与竞争的一定是等待时间最长的线程，非公平的则不会保证。非公平实现中 tryAcquireShared 转而调用 nonfairTryAcquireShared 函数 

        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }

    当返回值小于 0 说明信号量不够了，没有申请到足够的信号量，这是执行 doAcquireSharedInterruptibly(arg) 方法。

        private void doAcquireSharedInterruptibly(int arg)
            throws InterruptedException {
            final Node node = addWaiter(Node.SHARED);
            boolean failed = true;
            try {
                for (;;) {
                    final Node p = node.predecessor();
                    if (p == head) {
                        int r = tryAcquireShared(arg);
                        if (r >= 0) {
                            setHeadAndPropagate(node, r);
                            p.next = null; // help GC
                            failed = false;
                            return;
                        }
                    }
                    if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                        throw new InterruptedException();
                }
            } finally {
                if (failed)
                    cancelAcquire(node);
            }
        }

    这个方法与重入锁的 acquireQueued 十分相似，都是在 for 循环中请求锁，或者阻塞。当请求到锁 return 然后就可以用户自己的业务逻辑了。

3. 释放信号量

        public final boolean releaseShared(int arg) {
            if (tryReleaseShared(arg)) {
                doReleaseShared();
                return true;
            }
            return false;
        }

        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

    与重入锁的释放逻辑十分相似，这里不再赘述，代码并没有特殊的检测，也就意味着只要是能持有 semaphore 实例的都能去直接增加信号量的值。

### 实例: 生产者消费者

    public class SemaphoreTest {
        public static Semaphore semp = new Semaphore(0);
        public static ReentrantLock lock = new ReentrantLock();
        public static int i = 0;
        public static Runnable consumer = ()->{
            while (true){
                try {
                    semp.acquire();
                    {
                        lock.lock();
                        System.out.println(Thread.currentThread().getName()+" consume : "+ i);
                        i--;
                        lock.unlock();
                    }
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        public static Runnable producer = () ->{
            while (true){
                try {
                    {
                        lock.lock();
                        i++;
                        System.out.println(Thread.currentThread().getName()+"  produce: "+i);
                        lock.unlock();
                    }
                    semp.release();
                    Thread.sleep(10000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        public static void main(String[] args) {
            List<Thread> consumers = Stream.generate(()->new Thread(consumer)).limit(4).collect(Collectors.toList());
            List<Thread> producers = Stream.generate(()->new Thread(producer)).limit(2).collect(Collectors.toList());
            consumers.addAll(producers);
            consumers.stream().forEach(t -> t.start());
            consumers.stream().forEach(t -> {
                try {
                    t.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
    }

输出结果 : 

    Thread-5  produce: 1
    Thread-4  produce: 2
    Thread-0 consume : 2
    Thread-2 consume : 1
    Thread-5  produce: 1
    Thread-4  produce: 2
    Thread-3 consume : 2
    Thread-1 consume : 1

产生一个产品增加加一个信号量，虽然同是有4个消费者线程在跑，但是可以看到每次产生完产品只有两个线程能去消费。同时，信号量并不能替代锁，如果在消费或者生产过程中不加锁对 i 的操作就可能发生一致性问题，可能会见到如下的结果：

    Thread-5  produce: 2
    Thread-4  produce: 2
    Thread-0 consume : 2
    Thread-1 consume : 2

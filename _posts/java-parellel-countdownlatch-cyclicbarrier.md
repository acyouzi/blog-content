---
title: CountDownLatch 和 CyclicBarrier
date: 2016-11-23 15:17:59
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- java
	- 并发
---

### CountDownLatch
CountDownLatch 用于让某一线程等待多个线程完成前置工作后再执行。

有以下几个方法可供使用：
* public void await()
* public boolean await(long timeout, TimeUnit unit)
* public void countDown()

#### 示例

    public class CountDownLatchTest {
        public static CountDownLatch end = new CountDownLatch(10);
        public static Runnable run = ()->{
            try {
                Thread.sleep(new Random().nextInt(10)*1000);
                System.out.println("check complete!!!");
                end.countDown();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        };

        public static void main(String[] args) throws InterruptedException {
            List<Thread> list = Stream.generate(()->new Thread(run)).limit(10).collect(Collectors.toList());
            list.stream().forEach(t->t.start());

            end.await();
            System.out.println("complete all");
        }
    }

主线程会等待其他十个线程全部调用 end.countDown(); 后再执行。

####  实现原理
使用共享锁实现，countDown() 方法调用 releaseShared，await 调用 acquireSharedInterruptibly 方法，主要的实现逻辑在 tryAcquireShared tryReleaseShared 中

#### tryAcquireShared 与 await 相关 
当状态不为0时一直阻塞。

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

#### tryReleaseShared 与 countDown 方法有关。
每次调用在原来的状态基础上 -1,直到等于0为止

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            // 当某个线程把 state 搞到 0 ，然后方法会返回 true ,上层的 releaseShared 方法会调用 doReleaseShared 去唤醒等待的线程。
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }

### CyclicBarrier
作用跟 countdownlatch 差不多，功能比 CountDownLatch 强大，可以循环使用，可以传入一个 Runnable 用于在等待的多个前置线程做完工作后调用。

#### 示例

    public class CyclicBarrierTest {
        public static CyclicBarrier cyclic = new CyclicBarrier(10,()-> System.out.println("CyclicBarrier await condition complete"));
        public static Runnable run = ()->{
            try {
                System.out.println(Thread.currentThread().getName());
                Thread.sleep(new Random().nextInt(10)*1000);
                cyclic.await();
                System.out.println(Thread.currentThread().getName());
                Thread.sleep(new Random().nextInt(10)*1000);
                cyclic.await();
            } catch (Exception e) {
                e.printStackTrace();
            }
        };

        public static void main(String[] args) {
            List<Thread> list = Stream.generate(()->new Thread(run)).limit(10).collect(Collectors.toList());
            list.stream().forEach(t->t.start());
        }
    }

#### 实现原理
使用重入锁配合 condition 实现

    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        // 获取锁
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
            // 当 count == 0 时，执行在使 count 到 0 的那个线程中执行传入的 Runnable 接口，唤醒然后唤醒所有 await 的线程
            // 重置 count 然后进行下一代的计数
            int index = --count;
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // 循环，阻塞
            for (;;) {
                try {
                    if (!timed)
                        // Condition 的 await 方法，会让出当前持有的锁。
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }





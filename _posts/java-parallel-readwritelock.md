---
title: 读写锁解读
date: 2016-11-22 21:43:31
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- java
	- 并发
---

### 读写锁
创建两把锁，读锁是共享锁，写锁是独占锁，读锁允许多个线程同时读，写锁只能自己持有。写写操作，读写操作之间需要互相等待。

读写锁基于 AbstractQueuedSynchronizer 类实现，其主要实现逻辑在以下两个方法中：

####  tryAcquireShared 用于读锁
    
    protected final int tryAcquireShared(int unused) {
        Thread current = Thread.currentThread();
        int c = getState();
        // 写锁，非本线程持有直接返回
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return -1;
        int r = sharedCount(c);
        // 判断对列中是不是有等待写锁的节点，然后 cas 竞争锁
        if (!readerShouldBlock() &&
            r < MAX_COUNT &&
            compareAndSetState(c, c + SHARED_UNIT)) {
            if (r == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
            }
            return 1;
        }
        return fullTryAcquireShared(current);
    }

1. 如果写锁已经被其他线程持有，返回失败。
2. 否则，通过 readerShouldBlock 判断是否应该阻塞，判断规则是：
        
        (h = head) != null &&  (s = h.next)  != null &&  !s.isShared() && s.thread != null;

    头结点不为空，头结点的下一个节点在等待写锁则返回 true。

3. 如果readerShouldBlock返回false，并且读锁数量没有超过最大值，并且cas 修改读锁状态成功则更新 read 计数器，除了第一个持有读锁的线程，其余的都放到本地 ThreadLocal 中。
4. 如果 2、3 步失败，调用 fullTryAcquireShared ，在这个方法中首先判断当前线程是不是写锁持有者，如果不是直接返回。
    否则当前系统没有写锁时调用 readerShouldBlock 判断需不需要阻塞，如果需要说明除还有其他线程在等待写锁，那就必须要结束读锁请求了。退出之前还需要清除一下 ThreadLocal 里的本地变量。
5. 还是 fullTryAcquireShared ，剩下最后一段，可能发生当前线程是持有写锁的线程，或者在tryAcquireShared中 cas 操作失败这种情况。如果当前线程是持有写锁然后他又获取到了读锁，这种情况叫做锁降级。

        final int fullTryAcquireShared(Thread current) {
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                // 有写锁，不是当前线程持有
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                } else if (readerShouldBlock()) {  // 对列中有等待写锁的线程
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // 1. 当前线程持有写锁，来申请读锁
                // 2. tryAcquireShared 中 cas 失败
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }

#### 锁降级
锁降级指的是持有写锁的线程又获得读锁的情况，锁降级并不会自动释放写锁，所以线程是同时持有读锁和写锁，线程需要分别显式的释放读锁和写锁。

#### 锁降级的应用场景
考虑当前线程持有写锁，已经完成写操作，虽然还需要相关数据但是接下来只会进行读操作，并且在读操作过程中不希望数据被改变，那么可以再次申请读锁，然后释放写锁。这时，如果等待对列的开头是等待读锁的请求就可以获取的到锁了，这样做可以提升系统效率。(ps:我猜的场景，不知道是不是这样用)

#### tryAcquire 用于写锁

    protected final boolean tryAcquire(int acquires) {
        Thread current = Thread.currentThread();
        int c = getState();
        int w = exclusiveCount(c);
        if (c != 0) {
            // c !=0 但是 w 是 c 的低16位，w==0 ，那么c的高 16 位一定不为0，也就是读锁不为0
            if (w == 0 || current != getExclusiveOwnerThread())
                return false;
            if (w + exclusiveCount(acquires) > MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            // Reentrant acquire
            setState(c + acquires);
            return true;
        }
        if (writerShouldBlock() ||
            !compareAndSetState(c, c + acquires))
            return false;
        setExclusiveOwnerThread(current);
        return true;
    }

1. 如果读计数器不为0或者写计数器不为0且不是持有写锁的线程会失败。
2. 如果当线程已经持有写锁，则写锁持有数 +1
3. 否则在读计数器为0且写计数器为0时，调用 writerShouldBlock 判断是否应该阻塞，在非公平锁中这个函数一直返回 false,然后就去 cas 操作参与锁竞争。
    
### 关于 state
读写锁把读锁和写锁的计数器放在一个 int 类型的 state 变量中，高16位是共享计数器，低16位是独占计数器。通过位移或按位与操作实现对每个计数器的访问。

    c >>> SHARED_SHIFT
    c & EXCLUSIVE_MASK

### 读写锁使用
下面代码中尝试使用锁降级，然而并没有看到多少性能提升。3个线程使用写锁+读锁，10个线程只使用读锁。

    public class ReadWriteLockTest {
        public static ReentrantReadWriteLock rwlock = new ReentrantReadWriteLock();
        public static Lock rlock = rwlock.readLock();
        public static Lock wlock = rwlock.writeLock();

        public static Runnable rwlockThread = () -> {
            for (int i = 0; i < 100; i++) {
                try {
                    wlock.lock();
                    System.out.println("wlock");
                    Thread.sleep(100);
                    rlock.lock();
                    wlock.unlock();
                    Thread.sleep(300);
                    System.out.println("rlock");
                    rlock.unlock();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        public static Runnable readThread = () -> {
            for (int i = 0; i < 100; i++) {
                try {
                    rlock.lock();
                    Thread.sleep(100);
                    System.out.println(i);
                    rlock.unlock();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        public static void main(String[] args) {
            List<Thread> thread = Stream.generate(()->new Thread(rwlockThread)).limit(3).collect(Collectors.toList());
            thread.addAll(
                    Stream.generate(()->new Thread(readThread)).limit(10).collect(Collectors.toList()));
            Collections.shuffle(thread);
            long time = System.currentTimeMillis();
            thread.parallelStream().forEach(t -> t.start());
            thread.parallelStream().forEach(t -> {
                try {
                    t.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
            System.out.println("time = " + ((System.currentTimeMillis() - time)/1000));
        }
    }


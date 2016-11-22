---
title: java-parallel-readwritelock
date: 2016-11-22 21:43:31
tags:
---

### 读写锁
创建两把锁，读锁是共享锁，写锁是独占锁，读锁允许多个线程同时读，写锁只能自己持有。写写操作，读写操作之间需要互相等待。

读写锁基于 AbstractQueuedSynchronizer 类实现，其主要实现逻辑在以下两个方法中：

####  tryAcquireShared 用于读锁
    
    protected final int tryAcquireShared(int unused) {
        Thread current = Thread.currentThread();
        int c = getState();
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return -1;
        int r = sharedCount(c);
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
4. 如果 2、3 步失败，调用 fullTryAcquireShared ，在这个方法中首先判断当前线程是不是写锁持有者，如果不是直接返回。否则当前系统没有写锁时调用 readerShouldBlock 判断需不需要阻塞，如果需要说明除还有其他线程在等待写锁，那就必须要结束读锁请求了，不然会发生死锁。退出之前还需要清除一下 ThreadLocal 里的本地变量。
5. 还是 fullTryAcquireShared ，剩下最后一段，可能发生当前线程是持有写锁的线程，或者在tryAcquireShared中 cas 操作失败这种情况。如果当前线程是持有写锁然后他又获取到了读锁，这种情况叫做锁降级。

#### 锁降级
锁降级并不会自动释放写锁，所以线程是同时持有读锁和写锁，线程需要分别显式的释放读锁和写锁。

#### tryAcquire 用于写锁
1. 如果读计数器不为0或者写计数器不为0且不是持有写锁的线程会失败。
2. 如果当线程已经持有写锁，则写锁持有数 +1
3. 否则在读计数器为0且写计数器为0时，调用 writerShouldBlock 判断是否应该阻塞，在非公平锁中这个函数一直返回 false,然后就去 cas 操作参与锁竞争。

    protected final boolean tryAcquire(int acquires) {
        Thread current = Thread.currentThread();
        int c = getState();
        int w = exclusiveCount(c);
        if (c != 0) {
            // (Note: if c != 0 and w == 0 then shared count != 0)
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
    
### 实例





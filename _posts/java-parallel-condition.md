---
title: java-parallel-condition
date: 2016-11-21 17:12:40
tags:
---

Condition 与重入锁相关联，其关系类似于 synchronized 与 wait notify 之间的关系。

### Condition 的方法
* public final void await() 使当前线程等待同时释放锁 
* public final long awaitNanos(long nanosTimeout) 有限时间的等待
* public final boolean awaitUntil(Date deadline) 等待，直到某个时间之前
* public final boolean await(long time, TimeUnit unit) 有限时间等待
* public final void awaitUninterruptibly() 等待，并且不响应中断

### 
---
title: java 并发基础
date: 2016-11-19 19:21:13
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- java
	- 并发
---
### 可见性
保证共享变量在线程 A 中修改了，线程 B 中立即能够知道

### 指令重排与有序性
指令重排是指在保证串行语义一致性的情况下对指令进行合理的调整，但是这种调整在多线程环境下严重问题。

假设线程 A 有如下指令：

    int a = 1
    flag = true

在经过指令重排后可能会变为：

    flag = true
    int a = 1

这是假设有一个依赖于 flag 来判断变量 a 是否赋值的线程 B 如下：

    if flag == true:
        use a

这时线程 B 有可能根据 flag 判断 a 已经有值了，但是实际读取时可能 a 还没有赋值。

#### 为什么要指令重排？
尽量减少指令流水线中断，更充分的利用cpu资源。

### 线程的状态
1. NEW     刚创建还没执行
2. RUNNABLE  执行状态
3. BLOCKED  等待获取锁的状态
4. WAITING  无线时间的等待，需要被唤醒。
5. TIMED_WAITING 有限时间的等待
6. TERMINATED  结束状态

### run 方法
线程可以继承 Thread 然后重写 Thread 的 run 方法来实现：

    Thread t = new Thread(){
        @Override
        public void run() {
            // 重新 run 方法
        }
    };

也可以通过实现 Runnable 接口：
    
    Thread t = new Thread(new Runnable() {
        @Override
        public void run() {
            // do something
        }
    });

这两者没有区别，在调用 t.start() 方法启动线程后会在新的线程中执行 Thread 的 run 方法，而默认的 run 方法就在调用 Runnable 实现的 run 方法。

### 线程退出
首先不能使用 stop 方法，stop 方法强制退出会导致一个不完整的逻辑。可以设置一个退出标志或者在线程中判断 isInterrupted():

    Thread t = new Thread(new Runnable() {
        public void run() {
            while (true){
                if (Thread.currentThread().isInterrupted()){
                    break;
                }
                // 业务逻辑
            }
        }
    });

然后在主线程中如果希望结束线程可以调用：

    t.interrupt();

相比于自己设置的退出标志，中断的功能更为强大，在 sleep 或者 wait 时中断也能起作用，当在主线程调用 interrupt() 时，如果被中断线程正处在 sleep 中，会中断 sleep 并抛出异常，同时中断标记位会被清除，所以需要在捕获异常出再次标记中断然后等待中断逻辑去处理。

    Thread t = new Thread(new Runnable() {
        public void run() {
            while (true){
                if (Thread.currentThread().isInterrupted()){
                    break;
                }

                try {
                    // sleep 被中断打断并抛出异常
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }
    });

### wait && notify 
wait notify notifyAll 属于 Object 类，所以任何对象都可以调用，但是这几个方法调用必须包含在 synchronzied 语句中。

    public static Object obj = new Object();
    public static void main(String[] args) {
        Thread t1 = new Thread(new Runnable() {
            public void run() {
                synchronized (obj){
                    try {
                        System.out.println(Thread.currentThread().getId());
                        obj.wait();
                        System.out.println(Thread.currentThread().getId());
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });

        Thread t2 = new Thread(new Runnable() {
            public void run() {
                synchronized (obj){
                    try {
                        System.out.println(Thread.currentThread().getId());
                        Thread.sleep(10000);
                        obj.notify();
                        System.out.println(Thread.currentThread().getId());
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                }
            }
        });
        t1.start();
        t2.start();
    }

当 notify 把等待的线程唤醒时仍然需要重新获取锁才能继续执行。wait 与 sleep 的区别在于 wait 会释放所持有的锁，sleep 不会。另外需要注意的是 notify 操作不会释放锁，如果我们希望 notify 之后立即执行 wait 线程则不能用这种方法，应该换成 CountDownLatch 实现。

### suspend && resume 
suspend 在挂起进程是不会释放任何锁资源，所以不推荐使用。

    Thread.currentThread().suspend();
    Thread.currentThread().resume();

### join
join()  or  join(long millis) 等待依赖线程结束。

调用:

    Thread t = new Thread();
    t.start();
    t.join();

实现：一直调用当前线程的 wait 方法直到所等待的线程死亡。

    if (millis == 0) {
        while (isAlive()) {
            wait(0);
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }

### yield
主动让出 cpu 资源，但是还会再次参与资源争夺。

    public static native void yield();

### volatile 关键字
volatile 告诉虚拟机这个字段是容易被多个线程改变，然后虚拟机会采取手段保证其对于多线程的可见性。但是 volatile 并不能保证复合操作的原子性。

    public class ThreadTest {
        public static volatile int i = 0;
        public static class RunnableImpl implements Runnable{
            @Override
            public void run() {
                for (int j = 0; j < 1000; j++) {
                    i++;
                }
            }
        }
        public static void main(String[] args) throws InterruptedException {
            Thread[] ts = new Thread[10];
            for (int j = 0; j < 10; j++) {
                ts[j] = new Thread(new RunnableImpl());
            }
            Arrays.stream(ts).forEach((Thread t) -> t.start());
            Arrays.stream(ts).forEach((Thread t) -> {
                try {
                    t.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
            System.out.println(i);
        }
    }

上面的程序虽然声明了 i 是 volatile 类型，但是因为 i++ 是复合操作，所以结果并不正确。

### 守护线程
守护线程一般是用来完成一些服务性的工作，当用户的工作线程结束时，守护线程无事可做自然也需要退出。这就是守护线程的意思，随着用户线程退出而退出。

    t = new Thread(new RunnableImpl());
    t.setDaemon(true);

### 线程优先级
Thread 中预设了三个优先级

    public final static int MIN_PRIORITY = 1;
    public final static int NORM_PRIORITY = 5;
    public final static int MAX_PRIORITY = 10;

我们可以使用 setPriority 来设置优先级

    t.setPriority(Thread.MAX_PRIORITY);

### 线程异常处理器
    
    public static class RunnableImpl implements Runnable{
        @Override
        public void run() {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            throw new RuntimeException("Test");
        }
    }
    public static class ExceptionHandle implements Thread.UncaughtExceptionHandler{
        public void uncaughtException(Thread t, Throwable e) {
            System.out.println( t.getName() + " --- down");
            e.printStackTrace();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(new RunnableImpl(),"Test");
        t.setUncaughtExceptionHandler(new ExceptionHandle());
        t.start();
    }

### synchronized
* 对给定对象加锁
* 对实例方法加锁，相当于对当前实例加锁
* 对静态方法加锁，相当于对当前类加锁

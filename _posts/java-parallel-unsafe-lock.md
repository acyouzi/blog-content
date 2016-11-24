---
title: LockSupport Atomic 包 和 Unsafe 类
date: 2016-11-24 15:28:46
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- java
	- 并发
---

### LockSupport 基本使用
unpark 操作调用之前一定要确保线程是可用的，调用 park 操作时如果 permit 可用会直接返回(在 park 之前调用 unpark 的情况)，否则线程挂起等待。

    public static Runnable run = ()->{
        System.out.println("park");
        LockSupport.park();
        System.out.println("unpark");
    };
    public static void main(String[] args) {
        Thread t = new Thread(run);
        t.start();
        LockSupport.unpark(t);
    }

1. LockSupport 不能实例化，调用 UNSAFE 类实现功能
2. UNSAFE.unpark 操作不太安全，如果在线程没有启动的时候就调用 UNSAFE.unpark 实际上不会起任何效果，这时如果线程执行调用了 LockSupport.park 就只能一直锁在那里了。


### Atomic 包使用
1. 基本类型原子更新， AtomicBoolean AtomicInteger AtomicLong ，用法示例：
    
        AtomicInteger i = new AtomicInteger(0);
        i.compareAndSet(i.get(),1);
        System.out.println(i.get());
        System.out.println(i.getAndIncrement());
        System.out.println(i.getAndAccumulate(3,(x,y)->x+y));
        System.out.println(i.get());

2. 数组类型原子更新，AtomicIntegerArray AtomicLongArray AtomicReferenceArray，用法示例：

        int[] i = new int[] {0,0,0};
        AtomicIntegerArray ai = new AtomicIntegerArray(i);
        ai.compareAndSet(0,0,3);
        ai.compareAndSet(1,0,2);
        ai.compareAndSet(2,0,1);
        System.out.println(ai.get(0) +","+ ai.get(1) +","+ ai.get(2));

    创建对象时传入数组实际上做了 this.array = array.clone() 这样一个操作，所以 AtomicIntegerArray 的操作并不会影响原来的数组。

3. 引用类型原子更新，包括：AtomicReference  AtomicMarkableReference AtomicStampedReference

        public static class ASRTest{
            public int i;
            public ASRTest(int arg) {
                this.i = arg;
            }
        }
        public static void main(String[] args) throws Exception {
            ASRTest asr = new ASRTest(0);
            AtomicStampedReference stampedReference = new AtomicStampedReference(asr,0);
            stampedReference.compareAndSet(asr,new ASRTest(10),0,1);
            System.out.println(stampedReference.getStamp());
        }


4. 对象字段原子更新，包括：AtomicIntegerFieldUpdater AtomicLongFieldUpdater AtomicReferenceFieldUpdater

        public static class ASRTest{
            public volatile Integer i;
            public ASRTest(int arg) {
                this.i = arg;
            }
        }
        public static void main(String[] args) throws Exception {
            AtomicReferenceFieldUpdater updater = AtomicReferenceFieldUpdater.newUpdater(ASRTest.class,Integer.class,"i");
            ASRTest test = new ASRTest(0);
            updater.compareAndSet(test,0,10);
            System.out.println(test.i);
        }



#### cas 操作实现
Atomic 底层也是使用 Unsafe 的方法，所有方法的实现都与如下的代码类似。
    
    public class LockSupportTest {
        private volatile int state = 0;
        public void print(){
            System.out.println(state);
        }
        public static void main(String[] args) throws Exception {
            Field filed = Unsafe.class.getDeclaredField("theUnsafe");
            filed.setAccessible(true);
            Unsafe unsafe = (Unsafe) filed.get(null);

            long offset = unsafe.objectFieldOffset(LockSupportTest.class.getDeclaredField("state"));
            LockSupportTest test = new LockSupportTest();
            test.print();
            unsafe.compareAndSwapInt(test,offset,0,3);
            test.print();
        }
    }

### Unsafe 还能做的事情
上面的 park/unpark 和 cas 是 Unsafe 类应用在多线程同步例子，另外 Unsafe 类还能完成很多其他的功能。

1. 直接操作内存，putxxx getxxx putXxxVolatile getXxxVolatile 
2. 申请释放内存 allocateMemory reallocateMemory freeMemory copyMemory 

        Field filed = Unsafe.class.getDeclaredField("theUnsafe");
        filed.setAccessible(true);
        Unsafe unsafe = (Unsafe) filed.get(null);

        long addr = unsafe.allocateMemory(32*8);
        unsafe.putInt(addr,15);
        System.out.println(unsafe.getInt(addr));
        unsafe.freeMemory(addr);

3. defineClass defineAnonymousClass allocateInstance ensureClassInitialized
4. loadFence storeFence fullFence 用于防止指令重排造成的问题
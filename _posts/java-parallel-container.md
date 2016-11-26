---
title: 线程安全容器
date: 2016-11-26 12:12:17
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- java
	- 并发
---

### 线程安全容器 
java.util.concurrent 包下提供了一堆线程安全的容器：
   
    ConcurrentHashMap
    ConcurrentSkipListMap 跳表实现的 map
    ConcurrentLinkedDeque 双端队列
    ConcurrentLinkedQueue 
    CopyOnWriteArrayList 在读多写少的情况下性能好于 Vector
    ConcurrentSkipListSet 
    CopyOnWriteArraySet

### Collections.synchronizedxxx
也可以通过 Collections.synchronizedxxx 包装的到一个线程安全容器，如：
    
    Collections.synchronizedList(new LinkedList<String>());

这里实现线程安全的方法是对容器的每一个方法都用 synchronized 做同步，如：

    synchronized (mutex) {return list.set(index, element);}

### CopyOnWriteArrayList 的实现
CopyOnWriteArrayList 的主要是在 add set 等方法中更改链表时，通过 Arrays.copyOf 搞一个副本，在副本上修改，这样就不会影响到读取过程了，操作完成之后再用副本替换原来的链表。

    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }

### ThreadLocal
ThreadLocal 用于保存只有当前线程自己可以访问的数据，ThreadLocal 使用内部使用 HashMap 保存数据。

    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

以上代码中 getMap 或者 createMap 得到的是 ThreadLocal 的静态内部类 ThreadLocalMap, 这个类实现了 HashMap, 而且这个类保存的 Entry 实例是一个 WeakReference 的子类

    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

关于 WeakReference 参看：[译 · java reference objects](/2016/11/07/java-reference-objects/#Weak-References)

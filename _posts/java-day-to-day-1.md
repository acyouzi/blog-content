---
title: java 杂记(1)
date: 2016-11-19 15:44:31
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- java
---

### ArrayList 长度
ArrayList 使用数组存储，当空间不足时需要动态扩容。扩容是如下的逻辑：

    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

每次容量不足，扩大为原来的1.5倍。为了避免经常扩容导致效率低下，可以给数组赋一个较大的初始值 new ArrayList(100);

Vector 与 ArrayList 类似，但是可以指定扩容多大  Vector(int initialCapacity, int capacityIncrement)

### 不要在 finally 块中使用 return
在 finally 块中使用 return 会导致调用者捕获不到异常。
  
    public static int test() throws Exception{
        try {
            throw new Exception("throw exception");
        }finally {
            return 0;
        }
    }
    public static void main(String[] args) {
        try {
            test();
        }catch (Exception e){
            // catch 部分永远不会执行
            e.printStackTrace();
        }
    }

### Arrays.asList()
返回的是 java.util.Arrays.ArrayList , 是个静态内部类而不是 java.util.ArrayList 类。这个类没有实现 add 方法，数组是定长的，但是可以对数组中的元素修改、排序等。

### CPU 缓存伪共享问题
CPU 的三级缓存，读写是以行为单位的，一般缓存行大小是在 64字节。如果两个分别被单独线程频繁更新的变量在一个缓存行内，就会造成缓存失效。可以通过在一个变量周围填充一些字节占满整个缓存行。

    public static class A{
        long l1,l2,l3,l4,l5,l6,l7;
        public volatile long value = 0;
        long r1,r2,r3,r4,r5,r6,r7;
        public volatile long test = 0;
    }

经过测试，这种写法，在两个线程分别更新 value 和 test 变量时，速度比一般写法要快一倍还多。

### 对象排布规则
参考[这篇文章](http://www.importnew.com/1305.html)

首先对象头在32位下要占8个字节，在64位下，如果使用 UseCompressedOops 占 12 位，如果不使用占 16 位。
规则1：任何对象都是8个字节为粒度进行对齐的。
规则2：类属性按照如下优先级进行排列：长整型和双精度类型；整型和浮点型；字符和短整型；字节类型和布尔类型，最后是引用类型。这些属性都按照各自的单位对齐。

    1. 双精度型（doubles）和长整型（longs）
    2. 整型（ints）和浮点型（floats）
    3. 短整型（shorts）和字符型（chars）
    4. 布尔型（booleans）和字节型（bytes）
    5. 引用类型（references）

规则3：不同类继承关系中的成员不能混合排列。首先按照规则2处理父类中的成员，接着才是子类的成员。
规则4：当父类中最后一个成员和子类第一个成员的间隔如果不够4个字节的话，就必须扩展到4个字节的基本单位。
规则5：如果子类第一个成员是一个双精度或者长整型，并且父类并没有用完8个字节，JVM会破坏规则2，按照整形（int），短整型（short），字节型（byte），引用类型（reference）的顺序，向未填满的空间填充。

### 计算对象占内存大小
1. 通过 Instrumentation 类计算，在类中声明如下函数

        static Instrumentation inst;
        public static void premain(String args, Instrumentation instP) {
            inst = instP;
        }
        public static long getObjectSize(Object arg0){
            return inst.getObjectSize(arg0)
        }

2. 通过 UNSafe 类计算
    
        private static Unsafe unsafe;
        static {
            try {
                Field filed = Unsafe.class.getDeclaredField("theUnsafe");
                filed.setAccessible(true);
                unsafe = (Unsafe) filed.get(null);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        public static long getObjectSize( Object arg0 ){
            // Object[] array = new Object[]{arg0};
            // long baseOffset = unsafe.arrayBaseOffset(Object[].class);
            long maxoffset = 0;
            Class maxfieldClass = null;
            Class clazz = arg0.getClass();
            do {
                for( Field f : clazz.getDeclaredFields()){
                    if( !Modifier.isStatic(f.getModifiers())){
                        maxoffset = unsafe.objectFieldOffset(f);
                        maxfieldClass = f.getType();
                    }
                }
            }while ((clazz = clazz.getSuperclass()) != null );
            long last = (byte.class.equals(maxfieldClass)||(boolean.class.equals(maxfieldClass)))?1:( short.class.equals(maxfieldClass) || char.class.equals(maxfieldClass))?2: (long.class.equals(maxfieldClass)||double.class.equals(maxfieldClass))?8:4;
            maxoffset = maxoffset + last;
            long res = 0;
            if ( maxoffset % 8 != 0 ){
                res = (maxoffset/8)*8+8;
            }else {
                res = maxoffset;
            }
            return res;
        }

### 

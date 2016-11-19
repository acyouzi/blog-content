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
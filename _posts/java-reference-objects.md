---
title: 译 · java reference objects
date: 2016-11-07 22:08:15
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- java
	- References
---

### 原文链接
[java reference objects]( http://www.kdgregory.com/index.php?page=java.refobj )

### Introduction
balabala...

### java堆和对象声明周期
作为一个刚投身 Java 阵营的 C++ 程序员， 对于堆栈的理解很难转换过来。C++里面，对象可以用 new 运算符创建在堆上，也能通过自动分配创建在栈上。下面的情况在C++里面是合法的：在栈上创建一个 Integer 对象。但是用Java编译器去编译就会报错

    Integer foo = Integer(1);

Java 不同于C++，它把所有的对象存储在堆上，而且只能通过new 运算符去创建对象。栈上只存储对象的一个引用，而非对象本身。思考下面的代码

    public static void foo(String bar)
    {
        Integer baz = new Integer(bar);
    }

下面的图显示了这个方法中堆栈的关系，栈会划分成一个个的frames，frame里面保存了供方法使用的参数和本地变量。这些变量是指向堆中对象的指针。
![foo内存示例图]( img/stack_and_heap.gif )
现在来仔细看看foo()的第一行，这里new了一个Integer对象，jvm首先尝试去为这个对象寻找足够大的一块堆空间，在32位的JVM中大概需要12bytes。如果能够分配到足够的空间，接下来就会调用构造函数，构造方法中会解析传入的字符串并且初始化新分配的对象。最后jvm存储对象的指针到baz变量。

这是正常的情况，当然也会有不太正常的情况，比如当新的运算符不能为对象找到这12个字节空闲空间。这种情况下，在抛出OutOfMemoryError异常前，会进行一次垃圾收集操作尝试腾出一些空闲空间。

### 垃圾收集
当Java给出了在堆上创建对象的new操作符，但是没有给出对应的delete 云算符，当foo()方法返回时，已经超出了变量baz的作用范围，但是他指向的对象还在在堆上。如果这是故事的结束，那么所有的应用程序都会很快耗尽内存。所以Java提供了垃圾收集器去清理这些不再被引用的对象

当程序尝试创建新对象但是没有足够的堆空间时垃圾回收器就会开始工作。在收集器扫描堆，寻找并清理死亡对象的时候请求线程会被挂起。如果收集器不能清理出足够的空间，并且jvm无法扩展堆空间，创建对象就失败了，然后你的程序就挂了。

### Mark-Sweep
java中一个的垃圾收集器一直是个经久不衰的神话，很多了认为jvm使用引用计数算法来回收对象，但是实际上jvm使用的是扫描标记算法，扫描标记算法背后的原理很简单:每个不可达的队形都是垃圾并且应该被回收。

标记扫描算法的步骤如下:
Phase 1: Mark
垃圾收集器从root引用开始，遍历标记所有能到达的对象。
![Mark]( img/gc_mark.gif )

Phase 2: Sweep
所有没有被标记的对象就是不可达的，是垃圾。如果垃圾对象重写了finalizer方法，就把它加到finalization queue(后面会讲到)。若没有那么这块空间就可以回收了。(补充：这需要取决于具体的虚拟机实现)
![Sweep]( img/gc_sweep.gif )

Phase 3: Compact (optional)
一些垃圾收集器还有第三个步骤：紧凑。在这一步GC会移动对象合并那些垃圾收集遗留下来的空闲空间。这样可以防止内存碎片化，内存碎片会导致大段的连续内存分配失败。
![Compact]( img/gc_compact.gif )

So what are the "roots"? In a simple Java application, they're method arguments and local variables (stored on the stack), the operands of the currently executing expression (also stored on the stack), and static class member variables.

In programs that use their own classloaders, such as app-servers, the picture gets muddy: only classes loaded by the system classloader (the loader used by the JVM when it starts) contain root references. Any classloaders that the application creates are themselves subject to collection, once there are no more references to them. This is what allows app-servers to hot-deploy: they create a separate classloader for each deployed application, and let go of the classloader reference when the application is undeployed or redeployed.

It's important to understand root references, because they define what a "strong" reference is: if you can follow a chain of references from a root to a particular object, then that object is "strongly" referenced. It will not be collected.

So, returning to method foo(), the parameter bar and local variable baz are strong references only while the method is executing. Once it finishes, they both go out of scope, and the objects they referenced are eligible for collection. Alternatively, foo() might return a reference to the Integer that it creates, meaning that object would remain strongly referenced by the method that called foo().

Now consider the following:

LinkedList foo = new LinkedList();
foo.add(new Integer(123));
Variable foo is a root reference, which points to the LinkedList object. Inside the linked list are zero or more list elements, each of which points to its successor. When we call add(), we add a new list element, and that list element points to an Integer instance with the value 123. This is a chain of strong references, meaning that the Integer is not eligible for collection. As soon as foo goes out of scope, however, the LinkedList and everything in it are eligible for collection — provided, of course, that there are no other strong references to it.

You may be wondering what happens if you have a circular reference: object A contains a reference to object B, which contains a reference back to A. The answer is that a mark-sweep collector isn't fooled: if neither A nor B can be reached by a chain of strong references, then they're eligible for collection.


### Finalizers 

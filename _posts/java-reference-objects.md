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

那么，roots节点包括哪些呢？在一个简单的java application 中，方法参数，本地变量(存储在栈上)，正在执行的表达式的操作数(也存储在栈上)，静态类成员变量。

在例如app-servers等使用自己的类加载器的程序中，情况就不同了：只有使用系统 classloader(JVM启动时使用的类加载器)加载的类会包含在 root 引用中(翻译的不对？:only classes loaded by the system classloader (the loader used by the JVM when it starts) contain root references.)。任何程序自己创建的类加载的在没有应用的时候都会被收集。这也是app-servers能够热部署的原因：它们为每个已部署的应用程序创建一个单独的类加载器，并在卸载或重新部署应用程序时不再持有classloader引用

这对理解root引用很重要，它定义了什么是强引用：如果你能够找到一条从根节点到特定节点的引用链，那么这个对象就是有强引用的，不能被回收。

现在回到foo()方法，参数bar和本地变量baz在方法执行期间是强引用变量，当方法完成时，他们就超出了作用域，他们引用的对象就能被回收。或者考虑foo()可能会返回一个他创建对象的integer对象的实例，这意味着对象仍然保持着强引用。

考虑下面的代码：
    
    LinkedList foo = new LinkedList();
    foo.add(new Integer(123));

变量foo 是一个root引用，指向LinkedList对象，链表里面有0或多个元素，每个元素都执行它的后继。这是一个强引用链，这里的对象放的Integer是不能被回收的，
但是只要foo超出作用域，那这里面的所有东西就不再是强引用了。

你可能会想，如果发生了循环引用会怎样：答案是没关系...(扫面标记算法不怕这个)

### Finalizers 
C++ 允许对象定义析构函数：当对象超出作用域或者显式删除时析构函数会被调用来释放资源。对于多数对象，需要在这里释放new或者malloc申请的内存。

在java里面，垃圾收集器会替你处理内存清理，所以你不需要显式的声明析构函数。

但是，内存不是唯一需要释放的资源，比如FileOutputStream：当你创建一个对象实例，他会从文件系统申请一个句柄，如果让所有对流的引用在关闭之前超出范围，那么文件句柄会发生什么？
答案是流有一个finalize方法：一个由JVM在垃圾回收器回收对象之前调用的方法。在FileOutputStream的例子中，dinalizer会关闭stream,释放文件句柄刷新缓存确保所有的数据正确写到磁盘。

    /**
     * Cleans up the connection to the file, and ensures that the
     * <code>close</code> method of this file output stream is
     * called when there are no more references to this stream.
     *
     * @exception  IOException  if an I/O error occurs.
     * @see        java.io.FileInputStream#close()
     */
    protected void finalize() throws IOException {
        if (fd != null) {
            if (fd == FileDescriptor.out || fd == FileDescriptor.err) {
                flush();
            } else {
                /* if fd is shared, the references in FileDescriptor
                 * will ensure that finalizer is only called when
                 * safe to do so. All references using the fd have
                 * become unreachable. We can call close()
                 */
                close();
            }
        }
    }

任何对象都能有finalizer,你只需要重写finalize方法

    protected void finalize() throws Throwable
    {
        // cleanup your object here
    }

finalizers 方法看起来是一个简单的清理方法，但是他有一些严重的问题。首先你千万别依赖这个方法去做任何重要的事情，因为finalizer可能永远都不会执行，
程序可能在垃圾收集前就退出了，还有其他一些乱七八糟的问题。

### Object Life Cycle (without Reference Objects)
整体来看，对象的生命周期可以通过下面的简单图片来总结:创建，初始化，使用，可被收集，被回收。阴影区域表示对象“强可达”的时间.
![对象生命周期](img/object_life_cycle.gif)

### Enter Reference Objects
JDK 1.2 引入了 java.lang.ref 包, 和对象声明周期的三个新阶段: softly-reachable, weakly-reachable,和phantom-reachable. 这些阶段仅仅在对象回收时有用，并且所讨论的对象必须是引用对象的引用：
* softly reachablet 
    软引用，垃圾收集器会尽量保留这些对象，但是当内存不足将要抛出 OutOfMemoryError 错误时会先回收这些对象
* weakly reachable
    弱引用 WeakReference，垃圾收集器会随时回收，不会试图保留这种引用。实际上，对象会在老年代(major)垃圾收集中被回收，在新生代(minor)垃圾收集中可能会存活
* phantom reachable
    幻影引用？ PhantomReference，已经是选中被回收的对象了，并且finalizer方法已经运行(why?我不确定)，你没法通过get获得可访问的强引用对象。

正如你可能猜到的，将三个新的可选状态添加到对象生命周期图中会造成混乱。虽然文档表明从通过软，弱和幻影强烈可达到的逻辑扩展，但是实际要依靠你程序创建了什么样的引用对象。如果创建WeakReference但不创建SoftReference，那么对象将从强可达直接弱可打然后完成到收集。
![引用对象生命周期](img/object_life_cycle_with_refobj.gif)

It's also important to understand that not all objects are attached to reference objects — in fact, very few of them should be. A reference object is a layer of indirection: you go through the reference object to reach the referent, and clearly you don't want that layer of indirection throughout your code. Most programs, in fact, will use reference objects to access a relatively small number of the objects that the program creates.

### References and Referents

### Soft References

### Weak References

### Reference Queues

### Phantom References



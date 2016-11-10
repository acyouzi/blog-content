---
title: jvm垃圾收集与内存分配
date: 2016-11-05 16:35:56
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- jvm
	- 垃圾收集
---

### 对象还活着吗
这个问题是垃圾收集需要解决的第一个问题，一般来讲判断一个对象是否还能被访问到有如下两个方法：

### 引用计数算法
引用计数算法：通过给对象添加一个引用计数器，每当有一个地方引用计数器加一，引用失效时减一。

这种算法实现简单判定效率高，很多编程语言(如python)都采用这种这种算法进行内存管理，但是他很难解决对象之间相互循环引用的问题，所以java的主流虚拟机没有选用这种算法进行内存管理的

### 可达性分析算法
可达性分析算法：通过一些列称为GC Roots的对象作为起始点，从这些节点开始向下搜索，搜索走过的路径称为引用链，当一个对象到GC root没有任何引用链相连则判定为不可达。

GC Roots 对象包括：
* 虚拟机栈
* 方法区中静态属性引用对象
* 方法区中常量引用的对象
* 本地方法栈JNI引用的对象

### 强引用，软引用，弱引用，虚引用
关于强引用，软引用，弱引用，虚引用个深入的内容可以参看这篇文章 [java reference objects](/2016/11/07/java-reference-objects/)
强引用：只要引用还在永远不会被回收
软引用：由 SoftReference 类实现，在将要发生内存溢出之前会被回收
弱引用：由 WeakReference 类实现，只要垃圾回收工作就会被回收
虚引用：由 PhantomReference 类实现，唯一的作用就是能在这个对象被收集器回收时收到一个系统通知。

### 回收过程
1. 第一次标记：对象在进行可达性分析后发现没有与gc root相连，则进行第一次标记，然后判断需不需要执行finalize方法，如果对象没有覆盖finalize方法，或者finalize方法已经被调用过则被视为无需执行。
2. 如果需要执行finalize方法，则对象会被放到一个队列里面，并且在后面会被一个Finalizer线程去调用执行他。若在finalize方法中重新建立引用关系则可以避免被回收。
	
	实际上finalize方法并不保证一定会执行，最好忘记这个方法。
	// 下一步翻译一篇关于finalize的文章
3. 然后GC会对这个队列进行一个小范围的第二次标记，如果发现重新建立了引用关系则把对象移出待回收集合。

### 垃圾收集算法
垃圾收集算法主要有标记-清除算法、复制算法、标记整理算法、分代收集算法，其思想都比较直观容易理解。

1. 标记清除算法：
	顾名思义，分为标记和清除两个阶段，这种算法的问题一个是效率不高，另一个是内存碎片。
2. 复制算法：
	最简单的复制是把内存划分为容量相等的两块，每次使用其中一块，然后清理时把还存活的对象复制到另一块上面。

	商业虚拟机一般都会实现这种算法，但是并不是按照 1:1 的比例划分内存空间，而是将内存分为一块较大的 Eden 空间和两块较小的 Survivor 空间。当回收时，Eden 空间和一块 Survivor 空间存活的对象一次性的复制到另一块 Survivor 空间上。

	HotSpot虚拟机 Eden 和两个 Survivor 空间默认的比例是 8:1:1
3. 标记这里算法：
	从标记清除算法改进而来，其目的是通过整理过程避免出现内存碎片
4. 分代收集算法：
	根据对象活动周期不同把对象分为两类，每一类放在不同的内存块，采用不同的垃圾收集算法。
	一般会分为新生代和老年代，新生代的特点是会有大量对象生成和死去，对象生命周期短。老年代的特点是对象存活率比较高。

### 如何发起内存回收
发起内存回收首先要进行的就是对虚拟机进行可达性分析，在对虚拟机进行可达性分析的时候必须保证是在一个一致性的快照中进行，这也就导致了GC进行时必须停顿所有的java线程。对虚拟机进行可达性分析的优化中又产生了下边几个概念

#### 准确式GC与OopMap
准确式GC：当虚拟机进行全可达性分析时不需要扫描全部上下文还有全局引用，虚拟机有办法知道哪个地方存放了引用。，目前所有的主流虚拟机都采用准确式GC

OopMap：HotSpot中使用oopmap实现准确式GC，它会在在类加载完成时把对象的引用变量的偏移地址记录下来，放到一组成为OopMap的数据结构中。

#### 安全点与安全区域
HotSpot虚拟机没有为所有的指令都生成OopMap，只是在特定的位置记录这些信息，这些特定的位置就称为安全点。程序只有在安全点才能暂停进行GC，关于安全点的选定需要看指令是否具有长时间执行的特征。

在GC发生时如何让所有的线程都跑到安全点上再停顿？
分为抢占式和主动式两种方法，抢占式先把所有的线程全部中断然后再把没在安全点上的线程跑到安全点上，主动式是设置一个标志位，线程看到这个标志位自动运行到最近安全点挂起。

安全区域：在一段代码中引用关系不会发生变化，这个区域的任意地方GC都是安全的，线程执行到安全区域需要先标示自己进入安全区域，线程离开安全区域时要检查是否完成GC过程，如果没有则需要等待GC完成再离开。

### 垃圾收集器
先来看一下visualvm中能看到的内存情况，这里可以看到的是一个典型的分代收集算法的内存空间，把内存分为了新生代和老年代两代，新生代又分为Eden和S0 S1三部分。
![java默认内存空间](img/memory-visualvm.png)

我们可以在新生代和老年代采用不同的垃圾收集算法。

#### Serial收集器
单线程，收集速度慢，适用于Client模式下的新生代收集。可以设置的参数 
* -XX:+UseSerialGC 使用Serial + Serial Old的收集器组合进行内存回收
* -XX:SurvivorRatio 默认为8，代表Eden:Subrvivor = 8:1
* -XX:PretenureSizeThreshold 直接晋升到老年代对象的大小，设置这个参数后，大于这个参数的对象将直接在老年代分配

#### ParNew收集器
Serial的多线程版本，是许多Server模式下虚拟的首选算法，因为目前除了Serial收集器，只有它能与CMS收集器配合工作。

* -XX:+UseParNewGC 用ParNew + Serial Old的收集器进行垃圾回收
* -XX:+UseConcMarkSweepGC ParNew + CMS +  Serial Old的收集器组合进行内存回收，Serial Old作为CMS出现“Concurrent Mode Failure”失败后的后备收集器使用
* -XX:ParallelGCThreads	设置并行GC进行内存回收的线程数

#### Parallel Scavenge 收集器
关注吞吐量，是吞吐量优先的收集器。吞吐量=运行用户代码时间/(运行用户代码时间+垃圾收集时间)，so..垃圾收集时间减少当然是以减少每次回收的垃圾大小为代价。

* -XX:GCTimeRatio	GC时间占总时间的比列，默认值为99，即允许1%的GC时间
* -XX:MaxGCPauseMillis	设置GC的最大停顿时间
* -XX:UseAdaptiveSizePolicy	动态调整java堆中各个区域的大小以及进入老年代的年龄 其他收集器也能用

#### Serial Old 收集器
Serial 的老年代版本，可以作为CMS的后备方案

#### Parallel Old 收集器
可以配合Parallel Scavenge使用，在注重吞吐量的场合下可以考虑Parallel Scavenge + Parallel Old

* -XX:+UseParallelOldGC	使用Parallel Scavenge +  Parallel Old的收集器组合进行回收

#### CMS 收集器
以获得最短回收停顿时间为目标的收集器，非常适合在重视相应速度的服务器上使用。

CMS的收集过程分为以下四个阶段：
* 初始标记：会导致停顿，仅仅标记GC Roots 能直接关联到的对象，速度很快
* 并发标记：不会引发停顿，时间较长
* 重新标记：修成因为程序运作导致并发标记不准确的部分，会导致停顿。
* 并发清理：不会导致停顿

CMS 无法处理浮动垃圾，而且需要给并发运行的线程预留足够的老年代空间。如果CMS运行期间预留的内存无法满足程序需要会触发 Concurrent Mode Failure 错误，导致进行一次Serial Old 过程。

* -XX:CMSInitiatingOccupancyFraction 设置CMS收集器在老年代空间被使用多少后出发垃圾收集，默认值为68%，不宜设置的太高，太高会引发Concurrent Mode Failure
* -XX:+UseCMSCompactAtFullCollection 由于CMS收集器会产生碎片，此参数设置在垃圾收集器后是否需要一次内存碎片整理过程
* -XX:+CMSFullGCBeforeCompaction 设置CMS收集器在进行若干次垃圾收集后再进行一次内存碎片整理过程

#### G1 收集器
面向服务器端，具有如下特性：
* 能充分利用多cpu环境，缩短停顿时间
* 不需要与其他收集器配合，能采用不同的策略处理新生代老年代
* 整体上看采用标记-整理算法，局部基于复制算法实现，不会产生空间碎片
* 可预测停顿

G1收集器把堆分为很多大小相等的区域，虽然保留了新生代老年代的概念，但是新生代老年代不再是物理隔离的了。虚拟机启动参数设置为 -XX:+UseG1GC 然后从visualvm中可以看到g1的内存状态。
![g1内存布局](img/g1.png)

G1通过执行并发的全局标记来确定整个Java堆空间中存活的对象。标记阶段完成后，G1就知道哪些区域基本上是空闲的，然后在筛选回收阶段对各个区域的价值和回收成本进行排序，然后根据用户期望的GC停顿时间进行排序。

在jdk.18中，对内存中重复的字符串进行了优化，使他们指向同一个字符数组，可以使用 -XX:+UseStringDeduplication启用

如果应用追求低的停顿，G1可以尝试一下，如果追求高吞吐，G1可能并不会带来什么提升。

### GC日志
* -XX:+PrintGC 输出GC日志
* -XX:+PrintGCDetails 输出GC的详细日志
* -XX:+PrintGCTimeStamps 输出GC的时间戳（以基准时间的形式）
* -XX:+PrintGCDateStamps 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）
* -XX:+PrintHeapAtGC 在进行GC的前后打印出堆的信息
* -Xloggc:../logs/gc.log 日志文件的输出路径

以 [GC 开头 Young area 回收(Minor GC)
以 [Full GC  对老年代和新生代都进行回收
以 [Full GC(System)  代表调用System.gc() 产生

下面来看一下这段程序产生的GC日志

	// -XX:+PrintGCDetails -Xms20M -Xmx20M -Xmn10M -XX:+PrintHeapAtGC
	public class GCTest {
		public static void main(String[] args) {
			byte[] b1,b2,b3,b4;
			b1 = new byte[2 * 1024 * 1024 ];
			b2 = new byte[2 * 1024 * 1024 ];
			b3 = new byte[2 * 1024 * 1024 ];
			b4 = new byte[4 * 1024 * 1024 ];
			b1 = b2 = b3 = b4 =null;
			System.gc();
		}
	}

日志如下(经过删减)

	Heap before GC invocations=1 (full 0):
		PSYoungGen      total 9216K, used 6421K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
		ParOldGen       total 10240K, used 0K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
	[GC (Allocation Failure) [PSYoungGen: 6421K->1005K(9216K)] 6421K->3538K(19456K), 0.0027576 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
	Heap before GC invocations=2 (full 0):
		PSYoungGen      total 9216K, used 5258K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
		ParOldGen       total 10240K, used 6629K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
	[GC (System.gc()) [PSYoungGen: 5258K->839K(9216K)] 11888K->7469K(19456K), 0.0032592 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
	Heap before GC invocations=3 (full 1):
		PSYoungGen      total 9216K, used 839K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
		ParOldGen       total 10240K, used 6629K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
	[Full GC (System.gc()) [PSYoungGen: 839K->0K(9216K)] [ParOldGen: 6629K->1120K(10240K)] 7469K->1120K(19456K), [Metaspace: 4388K->4388K(1056768K)], 0.0183726 secs] [Times: user=0.01 sys=0.01, real=0.02 secs] 
	Heap after GC invocations=3 (full 1):
		PSYoungGen      total 9216K, used 0K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
		ParOldGen       total 10240K, used 1120K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)

可以看到第一次GC 是因为Allocation Failure ， b4 = new byte[4 * 1024 * 1024 ];这句想要在新生代申请内存但是 PSYoungGen total 9216K, used 6421K 空间不够了，所以发生担保分配。

第二次第三次是因为调用System.gc()导致，会先在新生代回收内存，然后再进行一次Full GC

### 内存分配策略
主要在新生代的Eden区上分配，如果启用了本地线程缓冲，那就优先在本地线程缓冲上分配，少数情况也会直接分配到老年代，具体分配策略取决于采用的垃圾收集器，还有虚拟机的相关参数设置。

对象如果在Eden出生，活过第一次 Minor GC 就会被移动到 Survivor 区域并且年龄增加1，然后每熬过一次 Minor GC 年龄就会加 1 ，当年龄增加到一定程度就会被移动到老年代中。

老年代晋升默认参数是15，可以通过如下参数调整
* -XX:MaxTenuringThreshold=

当然虚拟机也不是完全按照 MaxTenuringThreshold 处理晋升，如果 Survivor 空间中年龄相同的所有对象大小的总和大于 Survivor 空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代。

在发生 Minor GC 之前，虚拟机会检查老年代的最大可用连续空间是否大于新生代所有对象空间的总和，如果大于那么 Minor GC 可以确保是安全的，如果不成立虚拟机会查看 -XX:+/-HandlePromotionFailure 参数，如果运行担保失败，虚拟机会冒险进行一次 Minor GC ,如果不允许担保失败，会进行 Full GC。

为了避免Full GC 过于频繁，大部分情况下回允许保失败。


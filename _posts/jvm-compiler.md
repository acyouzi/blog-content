---
title: jvm编译器
date: 2016-11-16 12:38:56
author: "acyouzi"
tags:
	- jvm
	- 编译
---

### 早期优化
对法糖的处理

泛型擦除

    public static void main(String[] args) {
        Map<String,String> map = new HashMap<String,String>();
        map.put("hello","你好");
        System.out.println(map.get("hello"));
    }

上面的代反编译后变为如下形式

    public static void main(String[] args) {
        HashMap map = new HashMap();
        map.put("hello", "你好");
        System.out.println((String)map.get("hello"));
    }

泛型被移除了，也就是不管把什么作为泛型参数的类在jvm中都被当做同一个类。

泛型重载：因为泛型擦除所以如下方法在编译时会被判定为重复的。

    public static void method(List<String> list){
        System.out.println("String list");
    }
    public static void method(List<Integer> list){
        System.out.println("Integer list");
    }

自动装箱拆箱，遍历循环

    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1,2,3,4);
        for( int i : list){
            System.out.println(i);
        }
    }

反编译后的代码变为
    
    public static void main(String[] args) {
        List list = Arrays.asList(new Integer[]{Integer.valueOf(1), Integer.valueOf(2), Integer.valueOf(3), Integer.valueOf(4)});
        Iterator var2 = list.iterator();
        while(var2.hasNext()) {
            int i = ((Integer)var2.next()).intValue();
            System.out.println(i);
        }
    }

条件编译

    public static void main(String[] args) {
        if( true ){
            System.out.println("true block");
        }else {
            System.out.println("false block");
        }
    }

    // 编译为
    public static void main(String[] args) {
        System.out.println("true block");
    }

### 运行期优化
部分虚拟机在开始运行时是通过解释器进行解释执行，当虚拟机发现某个方法或者代码块的运行特别频繁时就把这段代码当做热点代码，然后通过即时编译器编译为本地平台相关代码。

即时编译器一般分为 Client Compiler 和 Server Compiler 简称C1,C2。c1 编译器只进行简单可靠的优化，c2 编译器会进行耗时较长的优化，甚至是一些激进的优化。我们可以通过 -client 或者 -server 参数去指定虚拟机运行在哪种模式下。

编译器与解释器共同运行的模式称为混合模式，我们可以通过参数 -Xint 强制虚拟机运行在解释模式，也可以通过参数 -Xcomp 强制虚拟机运行在编译模式

### 热点代码
多次被调用的方法
被多次执行的循环体

热点探测当方法目前主要有：

1. 基于采样点的热点探测，周期性的检测各个线程的栈顶，如果发现某个方法经常出现在栈顶，那么这个方法就是热点方法。
2. 基于计数器的热点探测，虚拟机会为每个方法甚至是代码块建立计数器，统计方法的执行次数。

HotSpot采用第二种方法，他为每个方法准备了两类计数器：方法调用计数器，回边计数器。方法调用计数器的默认阈值在Client模式下是 1500 次，在Server模式下是 10000 次，这个阈值可以通过 -XX:ComplieThreshold 设置。另外还存在一个热衰减度，当执行一定时间计数器还没达到阈值计数器值会减半，可以通过 -XX:ConterDecay 参数来关闭热衰减。另外可以使用 -XX:CounterHalfLifeTime参数设置半衰周期。回边计数器没有热衰减过程。

使用 -XX:+PrintCompilation 参数来实时打印被编译为本地代码的方法名称。

### 编译优化技术

#### 方法内联
他是编译器最重要的优化手段之一，除了消除方法调用的成本之外，更重要的意义在于为其他优化手段建立良好的基础。对于非虚方法可以直接进行内联，对于虚方法编译期间可能根本没法确定应该使用哪个版本的方法。

为了解决虚方法内联的问题，首先引进了一种称为类型继承分析的技术，如果遇到虚方法通过类型继承分析确定是否只有一个版本的方法。如果是那也进行内联，但是这种优化属于激进优化需要预留一个逃生门，称为守护内联。

关于激进优化，除了内联，对于出现概率很小的隐式异常，使用概率很小的分支都可能被激进优化移除。

#### 逃逸分析
逃逸分析的基本行为就是分析对象动态作用域，如果一个变量在线程外被引用称为线程逃逸，如果一个变量在的作用域超出了方法范围称为方法逃逸。

基于逃逸分析可以做一下优化：

1. 栈上分配，确定对象作用域不会超出方法范围可以把对象分配在栈上，这样可以随栈被回收。(Java没有实现这项优化)
2. 同步消除，对于没有线程逃逸的对象不用进行线程同步。
3. 标量替换，把一个java对象拆散，根据程序的方位情况将使用的成员变量恢复成原始类型 来访问就叫标量替换。如果逃逸分析证明一个对象不会被外部访问，并且这个对象可以被拆散那么程勋真正执行时就可能不会真正创建这个对象。

-XX:+DoEscapeAnalysis 开启逃逸分析，-XX:+PrintEscapeAnalysis查看分析结果，-XX:+EliminateAllocations 开启标量替换，-XX:EliminateLocks 来开启同步消除，-XX:+PrintEliminateAllocation查看标量替换结果。
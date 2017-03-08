---
title: scala 学习 06 函数
date: 2017-02-20 01:08:41
tags:
    - scala
---
函数的的实现使用了 jdk1.7 中新引入的指令 invokedynamic, 其他的 invoke 指令在类加载的解析阶段就已经完成从符号引用到直接引用的替换，但是这条指令会等到实际调用时才进行解析。 java8 的 lambda 表达式调用也是使用这条指令。

# 函数式编程
1. 将方法转换为函数使用 func _ 这样的语法
2. 函数定义 val xxx = (xxx,...) => { ... }
3. 高阶函数是指接受其他函数作为参数的函数，或者将函数作为值返回

    // 接受函数类型参数
    def xxx(func: (String) => Unit, id: String) { func(name) }
    // 返回函数类型参数
    def xxx(msg: String) = (name: String) => println(msg + ", " + name)

4. 高阶函数可以自动推断出参数类型，而不需要写明类型
5. 对于只有一个参数的函数，还可以省去其小括号
6. 如果仅有的一个参数在右侧的函数体内只使用一次，则还可以将接收参数省略，并且将参数用 _ 来替代
7. 闭包, 跟 js 的闭包差不多
8. 在 scala 调用 java 代码时可以使用隐式转换转变为 java 需要的类型

## 基本使用
看下边的例子

    object Main {
      def showInf(func :(String) => Unit,name: String): Unit ={
        func(name)
      }
      def main(args: Array[String]): Unit = {
        showInf(x => println(x),"acyouzi")
      }
    }

上面的代码定义了一个高阶函数，这个函数接收一个函数，然后调用。运行上面代码在虚拟机启动参数中添加 -Djdk.internal.lambda.dumpProxyClasses 打印生成的内部类。然后反编译查看其实现原理。

showInf 方法对应到 java 代码

    public void showInf(Function1<String, BoxedUnit> func, String name) {
      func.apply(name);
    }

我们需要的函数参数被对应成了 trait ,这个 trait 与 java 的 Function interface 十分相似。而具体调用的地方被翻译成了如下几条指令

    14: aload_0
    15: invokedynamic #73,  0             // InvokeDynamic #0:apply:()Lscala/Function1;
    20: ldc           #75                 // String acyouzi
    22: invokevirtual #77                 // Method showInf:(Lscala/Function1;Ljava/lang/String;)V
    
注意 invokedynamic 指令，这条指令是在 java7 中新加入的,而且 java8 的 lambda 表达式调用就是使用这条命令实现的。所以 scala 的函数跟 java lambda 表达式实现上应该很相似。

invokedynamic 会根据方法签名去 BootstrapMethods 中拿到方法的一个代理对象的实例。而这个代理对象是通过 LambdaMetafactory.altMetafactory() 方法生成的。这个方法最终会调用  InnerClassLambdaMetafactory 的 spinInnerClass 方法使用 asm 库提供的方法拼装一个实现了对应接口的代理类，代理类内容如下：

    final class Main$$$Lambda$3 implements Function1, Serializable {
      private Main$$$Lambda$3() {
      }
      @Hidden
      public Object apply(Object var1) {
        return $anonfun$main$1$adapted((String)var1);
      }
    }

在这个代理类中会调用一个静态的适配器方法，然后返回一个 BoxedUnit.UNIT

    com/acyouzi/scala/Main$.$anonfun$main$1$adapted((String)var1)

在这个方法中会调用真正的业务逻辑实现方法

    com/acyouzi/scala/Main$.$anonfun$main$1:(Ljava/lang/String;)V

这就是 scala 函数的调用流程。

这里还有一个问题，showInf 接受的函数被对应到了 Function1 接口，那么如果一个方法接受的函数对应不到某个接口该怎么办呢？可以看一下 scala 包下的相关 trait ,从 0 一直到 22 也就是最多可以对应到一个接受 22 个参数的函数。至于再多了什么情况我我懒得试了。

在 scala 包下边还有 Tuple 类 22个，从1 到 22，也就是说 tuple 里边应该最多只有 22个元素，同时每个 tuple 对应一个 Product 所以也就有 1 到22 个 Product

## 下划线使用
下划线跟在一个方法后面可以把方法转换为函数。

    val func = myfunc _
 
下划线在 lambda 表达式类型自动推导中可以简化书写，就是一些语法糖，没什么好注意的，会用就行了。

    val map = Map("e"->1,"a"->1,"d"->1,"c"->1,"b"->1)
    map.map(_._1).foreach(println(_));

## 闭包
函数在变量不处于其有效作用域时，还能够对变量进行访问，即为闭包。示例如下

    object Main {
      def test(): (String) => Unit ={
        var say = "Hi!"
        return (name:String) => println(s"$say $name")
      }
      def main(args: Array[String]): Unit = {
        var hello = test()
        hello("acyouzi")
      }
    }

闭包能够访问到超出作用域的变量，在 java 中是因为生成的代理类的实例中持有相关变量的引用。

实际上闭包被翻译成了一个对象，这个对象继承自 AbstractFunction，类名中带有 $anonfun$，引用的外部变量在构造函数中传入，具体业务逻辑放到 apply 方法中。

    public final class Main$$anonfun$test$1 extends AbstractFunction1<String, BoxedUnit> implements Serializable {
        public static final long serialVersionUID = 0L;
        private final ObjectRef say$1;
        
        public final void apply(final String name) {
            Predef$.MODULE$.println((Object)new StringContext((Seq)Predef$.MODULE$.wrapRefArray((Object[])(String[])new String[] { "", " ", "" })).s((Seq)Predef$.MODULE$.genericWrapArray((Object)new Object[] { (String)this.say$1.elem, name })));
        }
        
        public final /* bridge */ Object apply(final Object v1) {
            this.apply((String)v1);
            return BoxedUnit.UNIT;
        }
        
        public Main$$anonfun$test$1(final ObjectRef say$1) {
            this.say$1 = say$1;
            super();
        }
    }
    
而对方法的调用变成了如下的样子，注意 say 变量的类型变为了 ObjectRef 类型。

    public Function1<String, BoxedUnit> test() {
        final ObjectRef say = ObjectRef.create((Object)"Hi!");
        return (Function1<String, BoxedUnit>)new Main$$anonfun$test.Main$$anonfun$test$1(say);
    }
     
    public void main(final String[] args) {
        final Function1 hello = this.test();
        hello.apply((Object)"acyouzi");
    }

## Currying函数
科里化是一种将具备两个参数的函数 f 转化为使用一个参数的函数 g , 并且这个函数的返回值也是一个函数，他会作为一个新函数参数。

    f(x,y) = (g(x))(y)

科里化感觉像是闭包的一种特例，我也不太清楚这种理解对不对。实例如下

    object Main {
      def add(a: Int): (Int) => Int ={
        return (b: Int) => a + b
      }
      def main(args: Array[String]): Unit = {
        println(add(10)(15))
        var curry = add(10)
        println(curry(20))
      }
    }

scala 支持简写定义科里化函数，语法如下

    def xxx(x :Int)(y :Int) = x * y

## 控制抽象
把一系列语句归组成不带参数也没有返回值的函数

    def runInThread(block : => Unit): Unit ={
      new Thread(new Runnable {
        override def run(){
          block
        }
      }).start()
    }
    def main(args: Array[String]): Unit = {
      println(Thread.currentThread().getId)
      runInThread {
        println(Thread.currentThread().getId)
      }
    }

上面的方法采用换名调用方式声明参数( 换名调用：在参数声明和调用该参数的地方略去 () 但是保留 => .

这样调用的代码就能写成 xxx { .. } 的形式。这样用起来就像是我们自己定义了一个关键字在使用。

## 控制抽象配合闭包

    object Main {
      def mywhile (condition: => Boolean)(block: => Unit){
        if ( condition ){
          block
          mywhile(condition)(block)
        }
      }
      def main(args: Array[String]): Unit = {
        var i = 10
        mywhile( i > 0  ){
          println(i)
          i = i - 1
        }
      }
    }

上面 def mywhile (condition: => Boolean)(block: => Unit) 采用的是科里化的简写形式。如果不采用简写形式写出来的函数会让人崩溃的:

    object Main {
      def mywhile (condition: => Boolean): ( ()=> Unit) => Unit = (block : ()=> Unit) => {
        if(condition){
          block()
          mywhile(condition)(block)
        }
      }
      def main(args: Array[String]): Unit = {
        var i = 10
        mywhile( i > 0  )(()=>{
            println(i)
            i = i - 1
        })
      }
    }

上面是非简写形式，可以看到 mywhile 接受一个函数 ()=> Boolean, 返回一个接受函数的函数 ( ()=> Unit) => Unit， 在这个返回的接受函数参数的函数中实现了具体的业务逻辑。

## 常用高阶函数
1. map
    
        (1 to 9).map("*" * _).foreach(println _)

2. foreach

        (1 to 9).map("*" * _).foreach(println _)

3. filter 过滤掉不满足条件的
    
        (1 to 9).filter(_ %2 == 0).foreach(println(_)) 

4. zip 拉链操作

        (1 to 9).zip((1 to 9)).foreach(print(_))
        // 输出
        (1,1)(2,2)(3,3)(4,4)(5,5)(6,6)(7,7)(8,8)(9,9)

5. partition 根据条件，true 一组 false 一组

        val (x,y) = (1 to 9).partition(_%2 ==0)
        println(x)
        println(y)
        // 输出
        Vector(2, 4, 6, 8)
        Vector(1, 3, 5, 7, 9)

6. find 查找集合中第一个满足条件的元素，返回 option 类型，也就是可能为空

        (1 to 9).find(_ >5).foreach(println(_))

7. drop 删除前 n 个元素，类似还有 dropRight
 
        (1 to 9).drop(3).foreach(println(_))
    
8. dropWhile 将删除元素直到找到第一个匹配谓词函数的元素
    
        (1 to 9).dropWhile(_ % 2 != 0 ).foreach(print(_))
        // 输出
        23456789

9. foldLeft && flod 折叠
    
        (1 to 9).foldLeft(0){(m: Int, n: Int) => println("m: " + m + " n: " + n); m + n}
        // 输出
        m: 0 n: 1
        m: 1 n: 2
        m: 3 n: 3
        m: 6 n: 4
        m: 10 n: 5
        m: 15 n: 6
        m: 21 n: 7
        m: 28 n: 8
        m: 36 n: 9
    
10. foldRight 右折叠
    
        (1 to 9).foldRight(0){(m: Int, n: Int) => println("m: " + m + " n: " + n); m + n}
        // 输出
        m: 9 n: 0
        m: 8 n: 9
        m: 7 n: 17
        m: 6 n: 24
        m: 5 n: 30
        m: 4 n: 35
        m: 3 n: 39
        m: 2 n: 42
        m: 1 n: 44

11. flatten 扁平化
    
        List(List(1, 2), List(3, 4)).flatten.foreach(println(_))

12. flatMap 先 map 然后把结果扁平化
    
        List("this is a test sentence","this is another sentence").flatMap(_.split(" ")).foreach(println(_))

13. sortWith

        (0 to 9).sortWith(_>_).foreach(println(_))

14. groupBy

        (0 to 9).groupBy( _ %3 ).foreach(println(_))

15. reduce && reduceLeft && reduceRight && reduceOption

        var res = (1 to 9).reduce(_ + _)
        println(res)

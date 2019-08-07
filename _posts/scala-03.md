---
title: scala 学习 03 对象
date: 2017-02-20 00:51:57
description: 
tags:
	- scala
---
# object 
object，相当于class的单个实例，通常在里面放一些静态的field或者method

第一次调用object的方法时，就会执行object的constructor，也就是object内部不在method中的代码, 但是object不能定义接受参数的constructor,object的constructor只会在其第一次被调用时执行一次，以后再次调用就不会再次执行constructor了

object通常用于作为单例模式的实现，或者放class的静态成员，比如工具方法

## 单例原理
下面是我们定义的一个 Object 根据上面的介绍我们大体能猜出调用的执行结果

    object ObjectTest {
      println("ObjectTest")
      def print(): Unit ={
        println("wang wang wang!!!")
      }
    }
    
编译之后会得到两个 class 文件

    ObjectTest.class
    ObjectTest$.class

其中 ObjectTest.class 对应 java 代码如下

    public final class ObjectTest
    {
      public static void print()
      {
        ObjectTest..MODULE$.print();
      }
    }

这看起来更像是个代理类(仅在没有对应 class 的情况下)，而类的真正逻辑代码放在 ObjectTest$ 类中。

    public final class ObjectTest${
      public static  MODULE$;
      static{new ();}
      public void print()
      {
        Predef..MODULE$.println("wang wang wang!!!");
      }
      private ObjectTest$()
      {
        MODULE$ = this;Predef..MODULE$.println("ObjectTest");
      }
    }

首先构造函数是私有的，然后在 static 代码块中初始化创建类的实例。构造函数给静态公共变量 MODULE$ 赋值，这就是个简单而且标准的单例的写法。

可以使用 ObjectTest.print() 调用单例对象的方法， 当调用时候我们实际上调用的是代理类的静态方法，然后代理类的静态方法内调用具体实现类

## 伴生对象
如果有一个class，还有一个与class同名的object，那么就称这个object是class的伴生对象，class是object的伴生类

伴生类和伴生对象必须存放在一个.scala文件之中，类和伴生对象可以相互访问 private field，下面是一个例子：

    class ObjectTest {
      private var c1 = 100
      def doSomething(): Unit ={
        println("do something " + ObjectTest.o1)
      }
    }
    object ObjectTest {
      private val o1 = 200
      def print(): Unit ={
        var test = new ObjectTest()
        println("wang wang wang!!! " +  test.c1)
      }
    }

编译后也是生成两个文件

    ObjectTest.class
    ObjectTest$.class

ObjectTest.class 中的内容如下，可以看到 ObjectTest.c1 虽然是 private 类型，但是他的 get 方法变成了 public 方法。 我想这就是在伴生对象中能访问类的私有变量的原因吧

    public class ObjectTest
    {
      public int com$acyouzi$scala$ObjectTest$$c1()
      {
        return this.com$acyouzi$scala$ObjectTest$$c1;
      }
      
      private void com$acyouzi$scala$ObjectTest$$c1_$eq(int x$1)
      {
        this.com$acyouzi$scala$ObjectTest$$c1 = x$1;
      }
      
      private int com$acyouzi$scala$ObjectTest$$c1 = 100;
      
      public void doSomething()
      {
        Predef..MODULE$.println("do something " + ObjectTest..MODULE$.com$acyouzi$scala$ObjectTest$$o1());
      }
      
      public static void print()
      {
        ObjectTest..MODULE$.print();
      }
    }

根据上面的说明你一定以为伴生对象只能获取类的成员变量但是无法修改，因为毕竟 set 方法是 private.

然而事实总是出人意料，在 object ObjectTest 的 def print() 方法中加一句 test.c1 = 300 你会发现，竟然可以正确运行。然后反编译查看 java 代码，你会发现 set 方法竟然也变成 public 了

    public void com$acyouzi$scala$ObjectTest$$c1_$eq(int x$1)
    {
      this.com$acyouzi$scala$ObjectTest$$c1 = x$1;
    }
    
scala 还真是一门随便的语言。至于为什么类中能访问伴生对象的 private 对象，也是一样的套路，把 private 的 get/set 变成了 public

    public final class ObjectTest$
    {
      public static  MODULE$;
      private final int com$acyouzi$scala$ObjectTest$$o1;
      
      static
      {
        new ();
      }
      
      public int com$acyouzi$scala$ObjectTest$$o1()
      {
        return this.com$acyouzi$scala$ObjectTest$$o1;
      }
      
      public void print()
      {
        ObjectTest test = new ObjectTest();
        Predef..MODULE$.println("wang wang wang!!! " + test.com$acyouzi$scala$ObjectTest$$c1());
      }
      
      private ObjectTest$()
      {
        MODULE$ = this;this.com$acyouzi$scala$ObjectTest$$o1 = 200;
      }
    }

## object 继承抽象类
object也可以继承类，并覆盖类中的方法。

    object ObjectTest extends ConstructorTest{
      private val o1 = 200
      override def print(): Unit = {
        super.print()
        println("bbbbb")
      }
    }

## apply方法
通常会在伴生对象中实现 apply 方法，则当创建伴生类的对象时，可以不使用new Class的方式，而是使用Class()的方式，隐式地调用伴生对象得apply方法，这样会让对象创建更加简洁，例如 val arr = Array(1,2,3) 就是调用的 apply 方法

    class ObjectTest(var name: String,var age: Int){
    }
    object ObjectTest extends ConstructorTest{
      def apply(name: String,age: Int): ObjectTest = new ObjectTest(name,age)
    }
     
    // 调用
    val test = ObjectTest("acyouzi",23)
    
## update 方法
当你在 Array 对象中使用语法 arr(10) = 100 时实际上就是调用的 update 方法。

## unapply 方法
提取器，一般就是 apply 的反操作，在模式匹配的 case 中会用到

    case Test(a,b) => print(a + "  " + b)
    
## main方法
如果要运行一个程序，必须编写一个包含main方法类一样，scala中的main方法定义为def main(args: Array[String])。

    object Main {
      def main(args: Array[String]): Unit = {
        val test = ObjectTest("acyouzi",23)
        println(test.name +" --- "+ test.age)
      }
    }

前面介绍 object 编译会生成两个 class 文件，这里的 Main.class 中实现了实现了 java 的 main 方法。在 main 方法中会调用我们的单例 object 的 main 方法( public void main(String[] args) ) 

    public final class Main
    {
      public static void main(String[] paramArrayOfString)
      {
        Main..MODULE$.main(paramArrayOfString);
      }
    }

## 枚举
用object继承Enumeration类，并且调用Value方法来初始化枚举值

    object Season extends Enumeration {
      val SPRING, SUMMER, AUTUMN, WINTER = Value
    }
    // 或者传入枚举值的id和name
    object Season extends Enumeration {
      val SPRING = Value(0, "spring")
      val SUMMER = Value(1, "summer")
      val AUTUMN = Value(2, "autumn")
      val WINTER = Value(3, "winter")
    }
    // 访问
    Season(0)
    Season.withName("spring")
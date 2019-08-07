---
title: scala 学习 02 类基础
date: 2017-02-20 00:42:50
description: 
tags:
	- scala
---
# scala 类基础
下面是对 scala 类定义的一些基本概念以及对应到 java 语言中的一些思考

## 变量访问修饰,get/set 方法
下面先给出一段代码，通过这段代码可以了解一些类的基本特性

    class HelloScala {
      var a: Int = 1
      val b: Int = 2
      private var c: Int = 3
      private[this] var d: Int = 4
      var myname: String = "acyouzi"
      def name = "my name is " + this.myname
      def name_=(name:String){
        println("set new name")
        this.myname = name
      }
      def func1(i: Int)={ i }
      private def func2(i: Int): Int={ i }
    }
    
这段代码使用了 var,val private,private[this] 来定义类的成员，同时使用了自定义get/set方法的语法。下面给出对应的 java 代码

    public class HelloScala
    {
      public int a(){ return this.a;}
      public void a_$eq(int x$1){ this.a = x$1; }
      private int a = 1;
      
      public int b(){return this.b;}
      private final int b = 2;
      
      private int c(){return this.c;}
      private void c_$eq(int x$1){this.c = x$1;}
      private int c = 3;
      
      private int d = 4;
      
      public String myname(){return this.myname;}
      public void myname_$eq(String x$1){this.myname = x$1;}
      private String myname = "acyouzi";
      
      public String name(){return "my name is " + myname();}
      public void name_$eq(String name){
        Predef..MODULE$.println("set new name");
        myname_$eq(name);
      }
      
      public int func1(int i){return i;}
      private int func2(int i){return i;}
    }
    
通过上面两段代码对比能得出以下结论：
1. scala 的变量对应到 java 里面全部是 private 类型，自动生成 get set 方法访问，不管有没有 private 关键字修饰。
2. val 与 var 的区别是一个有 final 修饰，一个没有，并且 val 不会生成 set 方法(xx_$eq)
3. private 关键字作用在方法上，对于变量来说，如果使用 private 关键字修饰，其对应的 get set 方法是私有的。
4. 方法默认是 public，可以被 private 修饰。
5. private[this] 是作用是限定该变量是对象私有的，仅在当前对象中能够访问。从这里能够观察到的来看就是没有为变量生成 get/set方法，scala 外部对象访问类的成员变量都是通过 get/set 方法访问，找不到 get/set 也就没法从对象外部访问该成员变量了。
6. 自定义 get/set 方法，并不会影响默认生成的 get/set 方法。就像是给变量起了个别名
    
        def xxx = "my name is " + this.myname
        def xxx_=(name:String){} 
    
    注意 xxx_= 是 set 方法的名字，中间不要加空格，被翻译成 xxx_$eq.
7. 普通的类成员方法对应到 java 的方法，没什么需要注意的地方。

### private[this]
private[this] 是作用是限定该变量是对象私有的，仅在当前对象中能够访问，可能不太好理解，给出一个说明 private[this] 的例子。

    class TestPrivateThis {
      private[this] var name = "acyouzi"
      private var age = 99
      def print(test: TestPrivateThis): Unit ={
        println(s"$name -- $age")
        println(test.name+ " ---- "+ test.age)
      }
    }

上面的代码中 TestPrivateThis.name 变量是 private[this] 类型。print 方法执行时需要传入另一个 TestPrivateThis 对象，当编译时报错：

    TestPrivateThis.scala:11: error: value name is not a member of com.acyouzi.scala.TestPrivateThis
        println(test.name+ " ---- "+ test.age)
                     ^
    one error found

因为不在同一个对象中。

### 主构造函数
主constructor是与类名放在一起，类中没有定义在任何方法或者是代码块之中的代码，就是主constructor的代码，主constructor中还可以通过使用默认参数，来给参数默认的值

如果主constrcutor传入的参数什么修饰都没有，比如name: String，那么如果类内部的方法使用到了，则会声明为private[this] name；否则没有该field，就只能被constructor代码使用而已，如果有 var,val,private 之类的修饰,参数就是正常的类成员变量。

下面是几个例子：

    // scala 类参数没有修饰符，如果在类方法中没有使用就相当于函数的参数而已
    // 不会作为类的成员变量添加到类中
    class ConstructorTest(name: String, age: Int) {
      println(s"$name === $age")
    }
    // 对应 Java 代码
    public class ConstructorTest
    {
      public ConstructorTest(String name, int age)
      {
        // ... 
      }
    }

第二个是例子是添加修饰符的例子：

    class ConstructorTest(name: String, age: Int) {
      println(s"$name === $age")
      def print(): Unit ={
        println(s"$name === $age")
      }
    }

不知道什么原因用 Java Decomplier 反编译有些时候找不到对应的类成员，然后我尝试 javap -p -v 命令反编译字节码文件，在字段表中能找到这两个类的成员。

    private final java.lang.String name;
      descriptor: Ljava/lang/String;
      flags: ACC_PRIVATE, ACC_FINAL
     
    private final int age;
      descriptor: I
      flags: ACC_PRIVATE, ACC_FINAL

或者在编译的时候然编译器打印语法树也能看到他们的确变成了类成员

    scalac -Xprint:parse ConstructorTest.scala
    [[syntax trees at end of                    parser]] // ConstructorTest.scala
    package <empty> {
      class ConstructorTest extends scala.AnyRef {
        <paramaccessor> private[this] val name: String = _;
        <paramaccessor> private[this] val age: Int = _;
        def <init>(name: String, age: Int) = {
          super.<init>();
          ()
        };
        val i = 10;
        var j = 10;
        println(StringContext("", " === ", "").s(name, age));
        def print(): Unit = println(StringContext("", " === ", "").s(name, age))
      }
    }

### 辅助构造函数
辅助构造函数之间可以相互调用，而且对其他构造函数的调用必须放在第一行。

    def this(name: String, age: Int) {
      this(name)
      this.age = age
    }

### 内部类
这里的内部类对应于 Java 的成员内部类，按照成员内部类的特性来理解。

    class ConstructorTest(var name: String, private val age: Int) {
      class Test{
        val i = 10
      }
    }
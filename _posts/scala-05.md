---
title: scala 学习 05 trait
date: 2017-02-20 00:58:40
tags:
    - scala
---
我觉得我已经爱上 scala 了, 在 java 中很麻烦的事情感觉用 scala 几行代码就能搞定。比如 trait 的调用链. 简单分析了一下 trait 的调用链，真的是满满的语法糖，但是用起来简单了很多。scala 把一些常用的设计模式变成了语法特性,这会不会是未来语言设计的趋势呢？
# Trait
1. Scala中的Triat是一种特殊的概念,可以将Trait作为接口来使用,但是他的实际功能要比接口强大
2. 使用 extends 继承
3. scala不支持对类进行多继承，但是支持多重继承trait，使用with关键字即可
4. trait 还可以定义具体方法，当做工具类来使用
5. Triat 可以定义具体field，此时继承trait的类就自动获得了trait中定义的field。
6. Triat 可以定义抽象field，而trait中的具体方法则可以基于抽象field来编写
7. trait 可以继承 trait, 可以覆盖父trait的抽象方法的
8. trait 也是有构造代码的，就是trait中的，不包含在任何方法中的代码
9. trait 没有接受参数的构造函数，
10. trait 也可以继承自class，此时这个class就会成为所有继承该trait的类的父类

## 简单使用
有一个 logger trait 里面有定义的方法

    // trait 
    trait Logger {
      val name = "Logger"
      var test = "Logger"
      var describe: String
      def print(mess: Any)
      def showInf(): Unit ={
        println(s"$name : $describe")
      }
    }
    // Cola
    class Cola extends Drink with Goods with Logger{
      override def takeDrink(): Unit = {
        print("drink cola")
      }
      override var describe: String = "this is cola"
      override def print(mess: Any): Unit = {
        println("test : " + mess)
        showInf()
      }
    }

实际上 trait 对应到 java 里面就是 java8 的 interface, java8 interface  中可以添加使用 default 关键字修饰的非抽象方法, 还可以在接口中定义静态方法。下面给出上面代码的反编译结果

    // Logger
    public abstract interface Logger
    {
      public static void $init$(Logger $this){
        $this.com$acyouzi$scala$Logger$_setter_$name_$eq("Logger");
        $this.test_$eq("Logger");
      }
      public void showInf(){
        Predef..MODULE$.println(new StringContext(Predef..MODULE$.wrapRefArray((Object[])new String[] { "", " : ", "" })).s(Predef..MODULE$.genericWrapArray(new Object[] { name(), describe() })));
      }
      public abstract void print(Object paramObject);
      public abstract void describe_$eq(String paramString);
      public abstract String describe();
      public abstract void test_$eq(String paramString);
      public abstract String test();
      public abstract String name();
      public abstract void com$acyouzi$scala$Logger$_setter_$name_$eq(String paramString);
    }
    // Cola
    public class Cola extends Drink implements Goods, Logger{
      private String describe;
      private final String name;
      private String test;
     
      public void showInf(){ Logger.showInf$(this);}
      public String name(){return this.name;}
      public String test(){return this.test;}
      public void test_$eq(String x$1){this.test = x$1;}
      
      public void com$acyouzi$scala$Logger$_setter_$name_$eq(String x$1){
        this.name = x$1;
      }
      public void takeDrink(){ print("drink cola");}
      public String describe(){ return this.describe;}
      public void describe_$eq(String x$1){ this.describe = x$1; }
      public Cola(){
        Logger.$init$(this);
        this.describe = "this is cola";
      }
      public void print(Object mess){ .. }
    }

1. 首先可以看到 Logger 的确被声明成了接口，至于前面的 abstract, 字节码里面表示接口本来就有一个 abstract 标志，只不过我们平常在写 java 的时候默认给省略掉了。还有接口里边方法上的 abstract 也是这么会事。

2. Logger interface 里面有一个 public void showInf() 方法，这个方法就是 interface 的 default 方法。我们在trait 里面定义的方法被相当于 interface 的 default 方法

3. public static void $init$ 这个是 java interface 的 static 方法，这个方法用于对我们在 trait 中初始化过的变量初始赋值，估计所谓的trait 构造代码就是在这里面执行的。

4. trait 声明的变量最终都没有定义在 Logger Interface 中，而是直接落到了实现类中。interface 空留一堆get/set 的 abstract 接口。

5. trait val 类型变量的 set 抽象方法的名字变得很奇怪，因为val 不需被别人调用 set 方法，只需要他自己在初始化的时候调用一次就好了。

6. java Interface 里边的方法的访问修饰都是 public 的，但是 trait 里边却能够声明为其他的访问类型。如果把 field 声明为 private 感觉像是在实现类中给g该 field 换了个名字。

7. 发现一件很碉堡的事情，如果把 trait 里边非抽象方法的访问修饰符换成 private, 在接口的 class 文件中对应的方法真的声明为了 private 方法，这点就与 java 语言对应不起来了，java 里面default 函数也必须为 public 类型。这点应该就是 java != jvm 的一个例子吧。

8. 注意 trait 里边仅非抽象的 field、方法 能够使用访问修饰符。

## 继承
看下面的例子

    abstract class Drink {
      def takeDrink():Unit
    }
    trait Goods {
      def wang():Unit
    }
    trait Logger extends Drink with Goods{
      def log(): Unit
    }
    class Cola extends Logger{
      override def takeDrink(): Unit = {
        //
      }
      override def wang(): Unit = {
        //
      }
      override def log(): Unit = {
        //
      }
    }

logger 继承类 Drink 和 trait goods, Cola 继承 Logger,这种关系如果对应到 java 中变成了如下情况：

    public abstract interface Logger extends Goods {}
    public class Cola extends Drink implements Logger {}

因为 interface 不能继承类，所以对类的继承转移到实现类上。关于trait继承后方法的重写这里就不具体讨论了。

## 为实例混入trait

    // Logger 注意这里定义的方法不能是抽象方法，不然到实现类里面还的实现
    trait Logger {
      def print(any: Any): Unit = {}
    }
    // 覆盖默认的空方法
    trait MyLogger extends Logger{
      override def print(any: Any): Unit={
        println(s"MyLogger : $any")
      }
    }
    // 继承拥有空方法的 Logger 
    class Test extends Logger{
      def testTrait(): Unit ={
        print("test message")
      }
    }
    // 测试代码 创建对象时使用 whit 混入
    object Main {
      def main(args: Array[String]): Unit = {
        var test = new Test
        test.testTrait()
        test = new Test with MyLogger
        test.testTrait()
      }
    }

混入后实际调用的是 MyLogger 的实现相关实现方法，很神奇有木有。那么他到底是怎么实现的呢？请看下面代码

    Test test = new Test();
    test.testTrait();
    test = new Test()
    {
      public void print(Object any)
      {
        MyLogger.print$(this, any);
      }
    };
    test.testTrait();

有没有在心中飘过一万只 ... 竟然是个匿名类，匿名类里面覆盖了相关的接口。好甜的一颗语法糖啊。

## trait调用链
实现了设计模式里面的责任链。代码示例如下：
    
    // Handler 这里边的方法不能是抽象方法
    trait Handler {
      def handler(any: Any): Unit ={
      }
    }
    //  LogHandler.scala ，记得调用 super.handler(any) 传递数据
    trait LogHandler extends Handler{
      override def handler(any: Any): Unit = {
        println("print log : " +any)
        super.handler(any)
      }
    }
    //  AuthHandler
    trait AuthHandler extends Handler{
      override def handler(any: Any): Unit = {
        println("auth handler  " +any)
        super.handler(any)
      }
    }
    //  CheckHandler
    trait CheckHandler extends Handler{
      override def handler(any: Any): Unit = {
        println("print check : " +any)
        super.handler(any)
      }
    }
    // 继承多个 Handler 的实现接口，继承顺序的反向就是调用顺序
    // 先调用 CheckHandler
    class Test extends LogHandler with AuthHandler with CheckHandler{
      def doTest: Unit ={
        println("do test")
        handler("test handler")
      }
    }

scala 的这个特性感觉还是相当不错的，那么这具体是怎么实现的呢？请看下面分析

先给出 Test.class 和 其中一个 Handler 的实现类 LogHandler.class 反编译后实际的方法列表(javap 命令生成)

    public class com.acyouzi.scala.Test implements com.acyouzi.scala.LogHandler,com.acyouzi.scala.AuthHandler,com.acyouzi.scala.CheckHandler {
      public void com$acyouzi$scala$CheckHandler$$super$handler(java.lang.Object);
      public void handler(java.lang.Object);
      public void com$acyouzi$scala$AuthHandler$$super$handler(java.lang.Object);
      public void com$acyouzi$scala$LogHandler$$super$handler(java.lang.Object);
      public void doTest();
      public com.acyouzi.scala.Test();
    }
     
    public interface com.acyouzi.scala.LogHandler extends com.acyouzi.scala.Handler {
      public abstract void com$acyouzi$scala$LogHandler$$super$handler(java.lang.Object);
      public static void handler$(com.acyouzi.scala.LogHandler, java.lang.Object);
      public void handler(java.lang.Object);
      public static void $init$(com.acyouzi.scala.LogHandler);
    }

先看 LogHandler.class 多出来了一个奇怪的抽象方法以及一个静态方法

    public abstract void com$acyouzi$scala$LogHandler$$super$handler(java.lang.Object);
    public static void handler$(com.acyouzi.scala.LogHandler, java.lang.Object);

在这个静态方法中做的事情就是调用自己接口 handler 方法，而抽象方法的要等到在继承的类里实现了，Test.class 多出来的几个奇怪的方法就是我们 implements 的多个接口里边的抽象方法。
    
    public void com$acyouzi$scala$CheckHandler$$super$handler(java.lang.Object);
    public void com$acyouzi$scala$AuthHandler$$super$handler(java.lang.Object);
    public void com$acyouzi$scala$LogHandler$$super$handler(java.lang.Object);
    
Test.class 还重写了一个 public void handler(java.lang.Object) 方法，在这个方法里面调用 CheckHandler.handler$() 方法，也就是我们前边提到的静态方法。代码如下：

      public void handler(java.lang.Object);
        descriptor: (Ljava/lang/Object;)V
        flags: ACC_PUBLIC
        Code:
          stack=2, locals=2, args_size=2
             0: aload_0
             1: aload_1
             2: invokestatic  #28                 // InterfaceMethod com/acyouzi/scala/CheckHandler.handler$:(Lcom/acyouzi/scala/CheckHandler;Ljava/lang/Object;)V
             5: return
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
                0       6     0  this   Lcom/acyouzi/scala/Test;
                0       6     1   any   Ljava/lang/Object;
          LineNumberTable:
            line 6: 0
        MethodParameters:
          Name                           Flags
          any                            final

然后在 CheckHandler.handler$ 调用 CheckHandler.handler() 方法，也就是我们自己实现的 handler, 在这里面最后调用的 super.handler(any) 实际上被翻译成了调用 com$acyouzi$scala$CheckHandler$$super$handler 也就是接口自己的抽象方法。抽象方法的实现在 Test.class 中，代码如下：

      public void com$acyouzi$scala$CheckHandler$$super$handler(java.lang.Object);
        descriptor: (Ljava/lang/Object;)V
        flags: ACC_PUBLIC, ACC_SYNTHETIC
        Code:
          stack=2, locals=2, args_size=2
             0: aload_0
             1: aload_1
             2: invokestatic  #21                 // InterfaceMethod com/acyouzi/scala/AuthHandler.handler$:(Lcom/acyouzi/scala/AuthHandler;Ljava/lang/Object;)V
             5: return
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
                0       6     0  this   Lcom/acyouzi/scala/Test;
                0       6     1   any   Ljava/lang/Object;
          LineNumberTable:
            line 6: 0
        MethodParameters:
          Name                           Flags
          any                            final

可以看到着里边又去调用 AuthHandler.handler$ .. 然后这个责任链就连接起来了，注意是从最后继承的那个接口往前依次调用。最后调用到 LogHandler 的 com$acyouzi$scala$LogHandler$$super$handler，这个方法里面调用的是 com/acyouzi/scala/Handler.handler$ 而 Handler.handler() 方法是空，责任链结束。

## trait的构造机制
1. 父类的构造函数执行
2. trait的构造代码执行，多个trait从左到右依次执行(没什么原因，编译器就是这么搞的)
3. 构造trait时会先构造父trait，如果多个trait继承同一个父trait，则父trait只会构造一次
4. 所有trait构造完毕之后，子类的构造函数执行

## trait field的初始化
有下面几种写法，估计应该不太常用
    
    // 这种方法是在 Hello 中添加 field val name: String
    class Hello extends {
      val name: String = "test"
    } with FieldTrait{
      // ... 
    }
     
    // 这里用到了匿名内部类
    var test = new{
      val name = "test"
    } with Hello with FieldTrait

---
title: scala 学习 01 基本语法
date: 2017-02-17 01:52:17
description: 
tags:
	- scala
---
# 简述
文章是对最近正在学习的 scala 的一些总结，因为 scala 语法相比于传统编程语言有点变化太大了，所以为了更透彻的了解这些语法的特性，很多 scala 语法，我会在这一系列的 scala 学习文章中尝试把 scala 字节码翻译到 java 语言，以此来加深理解。

大概计划是 10 篇左右的文章，不知道啥时候能写完。

## 变量声明
val 声明常量,后续可以使用无法修改常量指向
    
    val xxx = xxx
    
var 声明变量，可以改变其引用
    
    var xxx = xxx
    
scala 中建议使用 val.

指定变量类型

    val name: String = xxx
    val name: Any = xx

如果是在方法体内部声明的局部变量，对应到 java 里面就是 java 的局部变量，val 与 var 在 java 代码上看来并没有什么不同

## 基本数据类型
    
    Byte,Char,Short,Int,Long,Float,Double,Boolean

scala 使用加强类对类的功能进行加强，如 StringOps, RichInt, RichDouble 等，scala 中的操作符 (+/-/*/&/||/!...) 都是函数。

注意 scala 中没有 ++ -- 方法

scala 中函数调用如果不需要传递参数，scala 允许调用时省略括号。

## 条件与循环
if 表达式有值，就是 if 或者 else 最后一行语句的执行值。
    
    val i = if(age > 30) 1 else 0 // 返回 1 或者 0
    
这个放到 java 中会被翻译成三元表达式，if 表达式可以自动推断类型。

    val j = if (i < 5 ) 0 else "Test"
    
上面这个表达式的值是 Any 类型，也就是 java 的 object 

    Object j = i < 5?BoxesRunTime.boxToInteger(0):"Test";
    
当然如果希望在 if else 语句中执行多句话，也可以使用 {}

    val j = if (i < 5 ) {
      println("oo1")
      1
    }else{
      println("oo2")
      2
    }

这样的语句就对应 java 中正常的 if else 表达式了。

    byte var10000;
    if(i < 5) {
      .MODULE$.println("oo1");
      var10000 = 1;
    } else {
      .MODULE$.println("oo2");
      var10000 = 2;
    }
    // 赋值语句在最后
    byte j = var10000;
    
只有 if 的没有 else 的表达式对应到java 中也需要出现 else 语句块，else 的返回值为 UNIT 类型。

    // scala 
    val j = if (i < 5 ) {
      println("oo1")
      1
    }
    // java
    if(i < 5) {
      .MODULE$.println("oo1");
      var10000 = BoxesRunTime.boxToInteger(1);
    } else {
      var10000 = BoxedUnit.UNIT;
    }
    Object j = var10000;
    
while 表达式还是跟老样子，但是有一点不同的是，条件里边只能放 boolean 类型，不会再把 int(0) 当做 false 了。

    var i = 10
    while ( i != 0 ){
      i -= 1
      print("Test")
    }

for 语句是不再是 java 中 for 语句的样式。对应到 java 中是一个 foreach 函数。  

    // scala 
    for (i <- 1 to 10 ) println(i)
     
    // java
    .MODULE$.to$extension0(scala.Predef..MODULE$.intWrapper(1), 10).foreach$mVc$sp((i) -> {
      scala.Predef..MODULE$.print(BoxesRunTime.boxToInteger(i - 1));
    });
    
多重循环，对应到 java 里边就是两个 lambda 外边是 i 里边是 j 

    for(i <- 1 to 9; j <- 1 to 9) {
      if(j == i) {
        println(i * j)
      } else if (i > j){
        print(i * j + " ")
      }
    }
    // java 代码，如果直接反编译看到的代码会引用很多自定义的类
    // 这里给出的是 java lambda 表达式的模拟实现
    Stream.iterate(1, i -> i + 1).limit(9).forEach(i -> {
      Stream.iterate(1, j -> j + 1).limit(9).forEach(j -> {
        if (i == j) {
          System.out.println(i * j);
        } else if (i > j) {
          System.out.print(i * j + " ");
        }
      });
    });
    
if 守卫 的使用

    // scala 
    for(i <- 1 to 100 if i % 2 == 0) println(i)
     
    // 对应到 java 中就是在 stream 中多了一个 filter
    Stream.iterate(1, i -> i + 1).limit(100).filter(i -> {
      return i % 2 == 0;
    }).forEach(i -> {
      System.out.println(i);
    });

yield 的使用

    // 用于构造集合
    val list = for(i <- 1 to 100 if i % 2 == 0) yield i;
     
    // java 实现
    Stream.iterate(1, i -> i + 1).limit(100).filter(i -> {
      return i % 2 == 0;
    }).map(i -> {
      return i;
    }).collect(Collectors.toList());
    
## 控制台读写
写：print,println,printf

读: scala.io.StdIn.readXxx 的一系列方法。

## 函数相关
使用 def 定义。必须给出所有参数的类型，但是不一定给出函数返回值的类型，只要右侧的函数体中不包含递归的语句，Scala就可以自己根据右侧的表达式推断出返回类型。

如果是递归函数需要明确给出返回值类型。单行函数，可以不使用 {}。

    def Func_Name([var_name: var_Type[,...]]) [[: return_Type] =] {
        // 函数体
    } 

例如 

    def fab(n: Int): Int = {
      if(n <= 1) 1
      else fab(n - 1) + fab(n - 2)
    }

### 默认参数
直接在参数声明后面 = "xxx"

    def Func_Name(var_name: var_Type = "xxxx"){
        // 函数体
    } 

### 带名参数调用
在调用时不按参数声明顺序，而是采取 var_name="xxx" 的形式调用,示例：

    // scala 
      def func(i :Int,j: String = "test",k: Boolean): Unit ={
        println(s"$i --- $j ---- $k")
      }
      def main(args: Array[String]): Unit = {
        func(k=false,i = 100)
      }
      
    // java 
      public void func(int i, String j, boolean k) {
        .MODULE$.println((new StringContext(.MODULE$.wrapRefArray((Object[])(new String[]{"", " --- ", " ---- ", ""})))).s(.MODULE$.genericWrapArray(new Object[]{BoxesRunTime.boxToInteger(i), j, BoxesRunTime.boxToBoolean(k)})));
      }
      public String func$default$2() {
        return "test";
      }
      public void main(String[] args) {
        boolean x$1 = false;
        byte x$2 = 100;
        String x$3 = this.func$default$2();
        this.func(x$2, x$3, x$1);
      }

通过上面的例子可以得出结论，这些玩意要是出现在 js 里面那就是被人吐槽的语法糖啊：
1. 默认参数不限制使用在参数列表的哪个位置，默认参数后面可以继续跟非默认参数，这点不同于常见的非默认参数
2. 带名参数调用，未带名参数必须在最前边, 在使用带名参数时默认参数仍然有效。
3. 默认参数翻译到 java 中保存在一个函数中，函数名是 funcname$default$index()
4. 带名参数，实际上是在编译的时候根据参数名对参数位置进行了调整。

### 变长参数

    def sum(nums: Int*) = {
      //
    }

实际上就是一个 seq 对象。

    public int sum(Seq<Object> nums) {
      // 
    }

所以在调用时可以 sum(1,2,3,4) 这样调用，如果有一个已经定义好的数组，必须的转换为 Seq 才能传入

    sum((1 to 10):_*)

### 过程
就是没有返回值的函数。定义函数时，如果函数体直接包裹在了花括号里面，而没有使用=连接，则函数的返回值类型就是Unit。这样的函数就被称之为过程。过程通常用于不需要返回值的函数，过程还有一种写法，就是将函数的返回值类型定义为Unit。

    def func2(i: Int){i}
    def func3(i: Int):Unit={i}

这些函数在 java 中的返回值会被设置为 void.

### lazy 值
如果将一个变量声明为lazy，则只有在第一次使用该变量时，变量对应的表达式才会发生计算。实际上就是搞了个函数。

    lazy val i = 10*10
    
### 异常处理
类似 java ，但是不特别区分受检查异常，因为很多触发异常的问题是无法在 catch 中解决的，所以有设计思想认为检查型异常非常不好，起不到实际作用但是增加了代码了冗余。

    try{
      throw new Exception("")
    }catch {
      case e: Exception => println(e)
    }finally {
      println("finally");
    }

## 包
scala 中的包和 java 中的包作用相同，同一个包可以定义在多个文件中，相关语法如下

    package com{
      package acyouzi{
        package scala{
          class Test{
            // do ...
          }
        }
      }
    }

包声明还可以写作串联式

    package com.acyouzi.test{
      class Person{
        // ...
      }
    }
或者像 java 一样进行文件顶部标记

## 包对象
包可以包含类，对象和 trait,但是如果我们希望添加一个工具函数，比较合理的做法是放到一个包下面，而不是添加到某个工具类中，这就是包对象的用途。

每个包都可以有一个包对象，需要在父包中定义它，并且名称与子包一样。

    package com.acyouzi
    package object scala {
      val xxx = "test"
    }

这样就能在整个包中直接使用包对象中定义东西了

## import
scala import 使用 * 作为通配符

    // 引入具体类
    import com.acyouzi.scala
    // 引入包下的全部类
    import com.acyouzi._
    // 引入类中的成员
    import com.acyouzi.scala._
    // 引入多个类
    import com.acyouzi.{Test1,Test2}
    // 重命名某个引入的类
    import com.acyouzi.{Test1 => Test}
    // 隐藏包下的某个类
    import com.acyouzi.{Test1 => _,_}

## 隐式引入
这三个包是默认引入的包：
    
    java.lang._
    scala._
    Predef._

## 文件读取

    val source = Source.fromFile("test.txt","UTF-8")
    // 按行读取
    val lineiterator = source.getLines()
    for (l <- lineiterator){
      println(l)
    }
    // 读取到一个数组里面
    val lines = source.getLines.toArray
    // 一次全部读取
    val content = source.mkString

## 从 URL 读取

    val source = Source.fromURL("http://www.acyouzi.com","UTF-8")
    println(source.mkString)

## 调用 shell 
需要导入相关包 import sys.process._

执行命令，结果打印到标准输出，然后执行是否成功(0,1)

    "ls -al ~" !

执行命令，返回输出结果( 两个感叹号 )

    "ls -al ~" !!

管道使用

    "ls -al ~" #| "grep test" !

输入输出流重定向

    "ls -al ~" #> new File("output.txt")
    "ls -al ~" #>> new File("output.txt")
    "grep xxx" #< new URL("http://xxx")

## 正则表达式

    // 生成 pattern
    var pattern = "[0-9]+".r
    // 当存在反斜杠时最好使用原始字符串语法，不需要对反斜杠进行转义
    var pattern = """[0-9]+""".r  
     
    // pattern 的常用方法
    pattern.findAllIn()
    pattern.findFirstIn()
    pattern.replaceAllIn()
    pattern.replaceFirstIn()
    
## 正则表达式组
把子表达式用括号括起来

    val pattern = """([0-9]+) ([a-z]+)""".r
    for( pattern(num,str) <- pattern.findAllIn("999 hello, 666 test")){
      println(s"$num   $str")
    }

## 拉链操作

    val a1 = Array("a", "b", "c")
    val a2 = Array(1, 2, 3)
    val map = a1.zip(a2)
    println(map.mkString("--"))
     
    // 输出
    (a,1)--(b,2)--(c,3)
    
## tuple
在 scala 存在 tuple1 - tuple22 这22个 tuple 对象，分别对应着含有 1 -22 个参数的 tuple. 

    var t = ("a","b","c")
    // 访问 tuple 元素
    println(t._1)


## 与 Java 集合互操作
scala 提供了很多集合隐式转换函数，通过引入相关隐式转换，能够简单的实现 java 集合和 scala 集合之间的相互转换。相关隐式转换函数在 scala.collection.JavaConverters 里面。


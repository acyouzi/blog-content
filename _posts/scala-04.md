---
title: scala 学习 04 继承
date: 2017-02-20 00:55:06
description: 
tags:
	- scala
---
# 继承
1. 使用 extends 关键字
2. 如果子类要覆盖一个父类中的非抽象方法，则必须使用override关键字
3. 子类可以使用 super 关键字显示的调用父类方法
4. 子类可以覆盖父类的val field，而且同时覆盖 get 方法，也需要使用 override 关键字
5. protected, protected[this] 用于父类和子类之间的访问控制。
6. 调用父类的constructor，只能在子类的主构造函数中调用，调用形式如下
    
        class Student(name: String, age: Int, var score: Double) extends Person(name, age) {
    
    注意！如果是父类中接收的参数，比如name和age，子类中接收时，就不要用任何val或var来修饰了，否则会认为是子类要覆盖父类的field
7. scala 中也能使用匿名内部类，而且貌似跟 java 差不多
8. 子类中覆盖抽象类的抽象方法时，不需要使用override关键字
9. 如果在父类中，定义了field，但是没有给出初始值，则此field为抽象field

## 简单示例
下面这个例子使用了 抽象方法，抽象field, 调用构造函数，在子类构造函数中覆盖 field. 

    // 父类
    abstract class Fruit(val weight: Int) {
      val name = "fruit"
      var prcie:Int
      def prePrint():Unit
      def print(): Unit ={
        prePrint()
        println(s"$name -- $prcie --- $weight")
      }
    }
    // 子类
    class Apple(weight:Int,override var prcie: Int) extends Fruit(weight){
      override val name = "apple"
      override def prePrint(): Unit = {
        println("I'm apple")
      }
    }

下面根据其对应java 代码来看一看其实现
使用 javap 反编译得到的代码显示中，父类字段表中有 weight,name 两个字段

      private final int weight;
        descriptor: I
        flags: ACC_PRIVATE, ACC_FINAL
     
      private final java.lang.String name;
        descriptor: Ljava/lang/String;
        flags: ACC_PRIVATE, ACC_FINAL

也就是说我们定义的抽象field var prcie:Int，压根就在父类中没留下痕迹，然后 price 的 get/set 方法却在在父类中被定义成了抽象方法

      public abstract int prcie();
        descriptor: ()I
        flags: ACC_PUBLIC, ACC_ABSTRACT
     
      public abstract void prcie_$eq(int);
        descriptor: (I)V
        flags: ACC_PUBLIC, ACC_ABSTRACT
        MethodParameters:
          Name                           Flags
          x$1  

也就时说这个抽象字段这个概念在 jvm 层面是不存在的。但是既然 scala 对 field 的访问都是通过 get/set 方法，那么 scala 里面把这个叫做抽象字段好像也很有道理。

下面是 class 文件反编译的到的java 代码(还是出现了字段表中有的字段，反编译的代码中竟然没有的情况)。

    // fruit
    public abstract class Fruit{
      private final int weight;
      public int weight(){ return this.weight;}
      public String name(){return this.name;}
      private final String name = "fruit";
      public abstract int prcie();
      public abstract void prcie_$eq(int paramInt);
      public abstract void prePrint();
      public void print(){ .. }
      public Fruit(int weight) {}
    }
    // apple
    public class Apple extends Fruit{
      public int prcie(){return this.prcie;}
      public void prcie_$eq(int x$1){this.prcie = x$1;}
      public Apple(int weight, int prcie){super(weight);}
      public String name(){return this.name;}
      private final String name = "apple";
      public void prePrint()
      {
        Predef..MODULE$.println("I'm apple");
      }
    }

## isInstanceOf和asInstanceOf
isInstanceOf判断出对象是否是指定类以及其子类的对象，不能精确判断出，对象就是指定类的对象
asInstanceOf将对象转换为指定类型

    val apple:Fruit = new Apple(100,100)
    apple.isInstanceOf[Fruit]
    apple.isInstanceOf[Apple]
    val test = apple.asInstanceOf[Apple]

## getClass和classOf
getClass可以精确获取对象的类，classOf[类]可以精确获取类，然后使用==操作符即可判断指定类的类型了。
    
    apple,getClass == classOf[Apple]


---
title: class文件格式实例解析
date: 2016-11-10 19:19:15
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- jvm
	- 类加载
---
### 概念
全限定名：包名.类名，然后把其中所有的 " . " 换成 " / " 最后再加一个分号,如com.acyouzi.test.Demo 的全限定名是 com/acyouzi/test/Demo;

描述符：用来描述字段的数据类型、方法的参数列表和返回值
* 基本类型和void用一个大写字符来表示
    B -> byte; C -> char; D -> double; F -> float; I -> int; J -> long; S -> short; Z -> boolean; V -> void; L -> 对象类型。
* 对于数组类型每一个维度使用一个前置的 " [ " 字符来描述。如 String[] 被标示为 [Ljava/lang/String 
* 描述方法按照先参数列表后返回值的顺序描述，参数列表按照参数的顺序放到一组小括号内。如 int func(char[] c,float b) 表示为 ([CF)I

### 字节码指令
Java 虚拟机采用的是面向操作数栈而非寄存器的架构，所以大多数的指令都是不包含在操作数的，只有一个操作码。 

### class 文件结构
class文件以8位字节为基础，当遇到需要占8位字节以上空间的数据项时，则会按照高位在前的方式分割成若干个8位字节进行存储。

1. magic number
    占4个字节，很多文件存储标准中都是用魔数来进行身份识别，Class文件的魔数是 0xCAFEBABE

2. 版本号
    5，6字节是次版本号，7，8字节是主版本号

3. 常量池
   * constant_pool_count 2 Byte 代表常量池容量，但是计数是从1开的，也就是说如果数值是35，那么实际上常量池中有34项常量。
   * constant_pool 有 constant_pool_count-1 项常量
    
    常量池中主要存放字面量和符号引用。字面量包括：文本字符串，声明为final的常量值等，符号引用包括：类和接口的全限定名，字段的名称和描述符，方法的名称和描述符

    java代码编译时没有连接的概念，而是在虚拟机加载class文件的时候动态的连接，所以class文件中没有保存方法字段的最终内存布局。当虚拟机运行时需要存需要从常量池中获得符号引用然后在翻译到具体内存地址中。

    常量池中每一项常量都是一个表，表的第一位是一个u1类型的标志位。后面的具体结构一句不同的标志位类型而不同，可以参考相关文档。class文件方法，字段等都需要引用 CONSTANT_Utf8_info类型的常量，so..这个类型能标示的最大长度就是（65535）就是最长的变量或方法名长度。

4. 访问标志 
    两个字节，标识是类还是接口，是否定义为public类型，是否定义为abstract类型，是否声明为final类型。

5. 类索引,父类索引，接口集合
    this_class  2 Byte
    super_class 2 Byte
    interfaces 一组 2 Byte 的集合，有一个 2 Byte的接口计数器

6. 字段表集合
    用于描述接口或者类中声明的变量，字段包括类级别变量和实例级别变量，但是不包括方法内部声明的局部变量。
    
    字段表由以下部分构成
   * 访问标志 2 Byte
   * name index
   * descriptor index
   * 属性表集合：由attributes_count和attributes组成，用于存储一些额外信息 

   字段表集合中不会列出从父类或者接口中继承来的字段，但是有可能列出原本Java代码中不存在的字段，例如内部类会持有外部类实例字段。

7. 方法表集合
    由以下部分构成
   * 访问标志 2 Byte
   * name index
   * descriptor index
   * 属性表集合：由attributes_count和attributes组成，用于存储一些额外信息     

   如果方法没有被重写，则子类中就不会出现父类的方法信息。实例构造函数是 < init > 方法，类构造函数是< clinit > 方法。

8. 属性表集合
   包括 Code属性，Constant Value属性，Deprecated属性，Exception属性等，下面列出了其中一些

9. Code属性
   * attribute_name_index 固定为指向常量池中的 Code 常量池
   * max_stack 操作数栈深度的最大值
   * max_locals 局部变量所需空间
   * code
   * exception_table 与后边的 exception属性 不同，格式: start_pc end_pc handler_pc catch_type
   * 等...

10. Exception 属性
    列举了方法中可能抛出的受检查异常

11. LineNumberTable 属性
    用于描述栈帧中的局部变量与用户自定义的名字之间的对应关系，不是运行时必须的，但是默认会生成到class文件中，可以通过 -g:none 或 -g:vars 来取消或要求生产这项信息。

    如果没有这项信息所有的参数名都将丢失，会给调试带来不便。
 
12. SourceFile属性
    非必须，记录源码文件名称

13. ConstantValue 属性
    作用是通知虚拟机为静态变量赋值，对于非静态变量，赋值时在实例构造器中赋值的，对于静态变量有两种选择：1，在类构造器中赋值。2，如果同时用 final static 修饰一个变量，并且数据类型是基本类型或者String类型就生成ConstantValue属性。

14. InnerClasses 属性

15. Deprecated 属性
    boolean 类型，表示某个类或方法字段，已经被推荐不再使用。@deprecated 注解

16. StackMap_Table 
    用于类型检查

17. Signature 属性
    记录泛型签名信息。


### 实例
下面看一下如下的类生成的字节码

    public class TestByteCode extends TestClass implements Test {
        public static int i = 1;
        public final static int j = 1;
        public int k = 1;

        class TestChild{
            public int t_c_i;
        }
        static {
            System.out.println("static");
        }
        public int function() throws Exception {
            try {
                this.k = 100;
            }catch (Exception e){
                System.out.println("error");
            }finally {
                System.out.println("finally");
            }
            return this.k;
        }
        public static int static_function() {
            return 2;
        }
    }   

    其中 TestClass 与 Test 定义分别为：
    public class TestClass {
        public int t_c;
    }
    public interface Test {
        public int t_i = 10;
    }

TestByteCode.class文件经过 javap -v 后生成的文件内容如下（经过删减），直接在内部添加注释说明关键点：

    public class com.acyouzi.reflect.TestByteCode extends com.acyouzi.reflect.TestClass implements com.acyouzi.reflect.Test
    // minor major version
    minor version: 0
    major version: 52
    // 访问标志
    flags: ACC_PUBLIC, ACC_SUPER
    // 常量池，总共62项
    Constant pool:
    // 第一项是实例构造函数，分别引用了常量池中11行和41行的内容，可以看到完整的方法签名构成
    #1 = Methodref          #11.#41        // com/acyouzi/reflect/TestClass."<init>":()V
    #2 = Fieldref           #10.#42        // com/acyouzi/reflect/TestByteCode.k:I
    .
    .
    .
    #24 = Utf8               Code
    #25 = Utf8               LineNumberTable
    #26 = Utf8               LocalVariableTable
    #27 = Utf8               this
    .
    .
    .
    #61 = Utf8               println
    #62 = Utf8               (Ljava/lang/String;)V
    {
    // 可以看到父类，接口的参数不会在class文件中体现出来
    public static int i;
        descriptor: I
        flags: ACC_PUBLIC, ACC_STATIC

    public static final int j;
        descriptor: I
        flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL
        // 前面介绍的关于 static 赋初值的两种方式之2，final + static + 基本数据类型或者String 
        ConstantValue: int 1

    public int k;
        descriptor: I
        flags: ACC_PUBLIC
    
    // 构造函数
    public com.acyouzi.reflect.TestByteCode();
        descriptor: ()V
        flags: ACC_PUBLIC
        Code:
        // 栈深度为2，局部变量所需空间1， args_size
        stack=2, locals=1, args_size=1
            0: aload_0
            // 执行父类的构造方法
            1: invokespecial #1                  // Method com/acyouzi/reflect/TestClass."<init>":()V
            4: aload_0
            5: iconst_1
            // 给非静态变量 k 赋值
            6: putfield      #2                  // Field k:I
            9: return
        // 参数名变量对应关系表
        LineNumberTable:
            line 3: 0
            line 6: 4
        // 本地变量，可以看到直接持有this指针
        LocalVariableTable:
            Start  Length  Slot  Name   Signature
                0      10     0  this   Lcom/acyouzi/reflect/TestByteCode;
    
    // 这部分注意异常情况
    public int function() throws java.lang.Exception;
        descriptor: ()I
        flags: ACC_PUBLIC
        Code:
        stack=2, locals=3, args_size=1
            0: aload_0
            1: bipush        100
            3: putfield      #2                  // Field k:I
            6: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
            9: ldc           #4                  // String finally
            11: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
            // 没有异常就跳到最后的 return 语句
            14: goto          48   
            17: astore_1
            18: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
            21: ldc           #7                  // String error
            23: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
            26: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
            29: ldc           #4                  // String finally
            31: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
            34: goto          48
            37: astore_2
            38: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
            41: ldc           #4                  // String finally
            43: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
            46: aload_2
            47: athrow
            48: aload_
            49: getfield      #2                  // Field k:I
            52: ireturn
        // 查表判断异常怎么处理，
        Exception table:
            from    to  target type
                // 0-6 行出现 Exception 类型异常要跳到 17 行执行处理逻辑
                0     6    17   Class java/lang/Exceptio
                0     6    37   any
                17    26    37   any
        // 用于类型检查
        StackMapTable: number_of_entries = 3
            frame_type = 81 /* same_locals_1_stack_item */
            stack = [ class java/lang/Exception ]
            frame_type = 83 /* same_locals_1_stack_item */
            stack = [ class java/lang/Throwable ]
            frame_type = 10 /* same */
        // 异常属性，列出了可能抛出的异常
        Exceptions:
        throws java.lang.Exception
    
    // 静态方法，不持有 this 指针
    public static int static_function();
        descriptor: ()I
        flags: ACC_PUBLIC, ACC_STATIC
        Code:
        stack=1, locals=0, args_size=0
            0: iconst_2
            1: ireturn
        LineNumberTable:
            line 25: 0
    
    // 构造代码块 是不是等价于 < clinit >?
    static {};
        descriptor: ()V
        flags: ACC_STATIC
        Code:
        stack=2, locals=0, args_size=0
            0: iconst_1
            <em>// static 变量 i 赋值初值</em>
            1: putstatic     #8                  // Field i:I
            4: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
            7: ldc           #9                  // String static
            9: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
            12: return
        LineNumberTable:
            line 4: 0
            line 12: 4
            line 13: 12
    }
    SourceFile: "TestByteCode.java"
    // InnerClass的内容
    InnerClasses:
        #14= #13 of #10; //TestChild=class com/acyouzi/reflect/TestByteCode$TestChild of class com/acyouzi/reflect/TestByteCode
 
### 总结
* final static 与 static 变量赋初值的位置，虽然在定义的位置就写出赋值，但是可能并不是声明完就立即赋初值，下面的非静态变量也是一样。
* this 指针在非静态方法中被默认添加到本地变量表
* 不管有没有写明，在构造函数第一句都会调用父类的构造方法
* 非静态变量赋初值会被移动到构造方法中，紧跟在父类构造方法调用语句之后
* 异常的处理是通过 exception_table
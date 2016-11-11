---
title: JVM 类加载阶段与类加载器
date: 2016-11-11 19:18:08
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
tags:
	- jvm
	- class
---

### 类的生命周期
从进入内存开始的生命周期包括：加载，验证，准备，解析，初始化，使用，卸载几个阶段。验证，准备，解析三个阶段合称为链接阶段

其中，加载，验证，准备，初始化，卸载这五个阶段的顺序是确定的，解析则顺序不确定。可能在初始化前，也可能等到真正执行时在进行解析，需要视情况而定。

#### 加载
加载时类加载的第一个阶段，在这个阶段虚拟机需要通过全限定名获得此类的二进制流，二进制流可以从包中获取，也可以从网络获取，计算机实时计算生成(动态代理)，从数据库读取，由其他文件生成(jsp)

加载完成后二进制流存储在方法区之中，然后会实例化一个Class类的对象，Class对象放在方法区(这一点与其他对象不同)。

#### 验证
验证阶段是为了保证类文件的合法性，大致会从四个方面进行验证：

1. 文件格式验证,这个阶段之后字节流才会进入方法区存储。
2. 元数据验证，主要是进行语义校验，比如：是否有父类，父类是否不允许继承，是不是抽象类，类中字段方法与父类是否矛盾。
3. 字节码验证，确定程序语义是否合法，符合逻辑。
4. 符号引用验证，对类自身以外的信息进行匹配性校验。

验证是个重要的阶段，但是并不是必须的，如果能确认类文件的安全，可以通过参数 -Xverify:none 来关闭大部分的校验措施，缩短类的加载时间。

#### 准备
正式为 static 变量分配内存并设置类变量初始值的阶段，这里的初始值指的是数据类型的 0 值，而非程序中指定的值。当然如果类字段的字段属性表中存在 ConstanValue 属性(static + final + 基本类型/String)，准备阶段就会根据 ConstanValue 给变量赋值。

#### 解析
解析阶段是虚拟机把常量池中的符号引用替换为直接引用的过程。直接引用是直接执行目标的中指针或者相对偏移量，或者能间接定位到目标的句柄。

对于同一个引用可能进行多次解析，除了 invokedynamic 指令之外其余的指令都是返回第一次解析的结果，避免重复解析。

对于 invokedynamic 指令，是一条用于支持动态语言的指令，要求等到实际运行到这条指令时解析动作才发生。而其余的指令都是可以在完成加载阶段就进行解析的。

#### 初始化方法
初始化时类加载过程的最后一步，到了初始化阶段，才开始真正执行类中定义的代码。这个阶段会执行 < clinit >() 方法。在多线程环境下虚拟机会保证 clinit 方法只执行一次。

< clinit >() 方法是由编译器自动收集类中所有类变量的赋值动作和静态语句块中的语句合并产生，其中代码的顺序与在程序中的顺序一致。静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值但是不能访问。

    public class StaticTest {
        static {
            i = 11;
        }
        static int i;
        public static void main(String[] args) {
            System.out.println(StaticTest.i);
        }
    }

    // 输出结果
    11

如果一个类中没有静态语句可以不生成 < clinit >() 方法。

< clinit >() 方法中没有显示的调用父类 < clinit > 方法，但是虚拟机会保证在子类的< clinit >方法调用之前父类的< clinit >已经被调用完成。

另外接口中虽然不能有 static 代码块，但是仍然可能有赋值语句，所以也需要 < clinit > 方法。

#### 被动引用
相对于遇到new,反射，子类初始化父类必须先初始化，用户指定的包含mian方法的主类等情况下必须立即进行初始化的情况，其余的引用类的场景不会触发初始化，称为被动引用。

例子：通过子类引用父类静态字段不会导致子类初始化
    
    public class FClass {
        static {
            System.out.println("FClass clinit");
        }
        public static int i = 11;
    }
    public class CClass extends FClass{
        static {
            System.out.println("CClass clinit");
        }
    }
    // 测试类，注意不要在CClass里面测试，如果这样成了必须进行初始化的情况中 用户指定的主类 这种情况了。
    public class Test {
        public static void main(String[] args) {
            System.out.println(CClass.i);
        }
    }

输出结果

    FClass clinit
    11

例子：常量在编译阶段会直接存入调用类的常量池，与原来的定义常量的类在Class文件层面没有半毛钱关系

    // 使用 -XX:+TraceClassLoading 参数查看类加载信息
    class Hello{
        static {
            System.out.println("Hello.class clinit");
        }
        public static final String HELLO = "wa haha...";
    }
    public class Test {
        public static void main(String[] args) {
            System.out.println(Hello.HELLO);
        }
    }

输出结果(部分)

    [Loaded com.acyouzi.classload.Test from file:/E:/IdeaProjects/reflect/out/production/reflect/]
    [Loaded sun.reflect.NativeMethodAccessorImpl from D:\Java\jdk1.8.0_101\jre\lib\rt.jar]
    [Loaded sun.reflect.DelegatingMethodAccessorImpl from D:\Java\jdk1.8.0_101\jre\lib\rt.jar]
    wa haha...

可以看到 Hello 类直接没有加载，下面看 javap -v 解析输出的class文件可以发现一些信息

    // 常量池中直接存储了常量字符串
    Constant pool:
        #4 = String             #25            // wa haha...
        #25 = Utf8               wa haha...
    // test main 方法中的 Code 部分，可以看到直接是使用了常量池中的一个常量，
    Code:
        stack=2, locals=1, args_size=1
            0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
            3: ldc           #4                  // String wa haha...
            5: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
            8: return

这种特性叫做常量传播优化

### 类加载器
类加载器做的工作是："通过类的全限定名来获取描述此类的二进制字节流"。

每一个类加载器都一个独立的类名称空间，在判断两个类是否相等时，只有这两个类是由同一个类加载器加载的前提下才有意义。否则就算两个类来自同一个class文件也是不相等的。

#### 类加载器类型
* 启动类加载器(Bootstrap ClassLoader),负责加载 <JAVA_HOME>/lib 目录或者被 -Xbootclasspath 参数指定的路径中，并且需要时虚拟机可识别的文件名。
* 扩展类加载器，负责加载 <JAVA_HOME>/lib/ext 目录下的东西，或者被 java.ext.dirs 系统变量所指定的路径，开发者可以直接使用扩展类加载器。
* 应用类加载器，或者叫系统类加载器，通过 ClassLoader.getSystemClassLoader 方法获取。一般这个就是程序中的默认类加载器。

#### 双亲委派模型
双亲委派模型的工作过程是：类加载器收到一个加载请求，首先把请求委派给父类加载器去完成，每一次的类加载器都是这么办，最后应该是顶层的启动类加载器首先尝试加载。当父加载器无法完成加载时，子加载器才会尝试自己去加载。






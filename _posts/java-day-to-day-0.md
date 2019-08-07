---
title: java 杂记(0)
date: 2016-11-16 16:38:50
author: "acyouzi"
description: 
tags:
    - java
---

### 动态代理的真面目

    public class DynamicProxyTest {
        interface IHello {
            void sayHello();
        }
        static class Hello implements IHello {
            @Override
            public void sayHello() {
                System.out.println("Hello every one");
            }
        }
        static class DynamicProx implements InvocationHandler {
            Object originClass;
            // 返回 代理对象，也就是下边的 $Proxy0 
            Object bind( Object origin ){
                this.originClass = origin;
                return Proxy.newProxyInstance(origin.getClass().getClassLoader(),origin.getClass().getInterfaces(),this);
            }
            @Override
            public Object invoke(Object prox, Method method, Object[] args) throws Throwable {
                System.out.println("uh .hahah");
                return method.invoke(originClass,args);
            }
        }
        public static void main(String[] args) {
            // 保存动态创建的代理类
            System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
            IHello h = (IHello)new DynamicProxy().bind( new Hello());
            h.sayHello();
        }
    }

使用这句 System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true") 设置保存代理类的class文件会保存一个$Proxy0的代理了类。代理类反编译代码如下： 

    final class $Proxy0 extends Proxy implements IHello {
        // 通过 Class.forName 得到类对应方法的引用。用于实际对象的反射 invoke
        private static Method m1;
        private static Method m3;
        private static Method m2;
        private static Method m0;

        public $Proxy0(InvocationHandler var1) throws  {
            super(var1);
        }

        public final boolean equals(Object var1) throws  {
            try {
                return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
            } catch (RuntimeException | Error var3) {
                throw var3;
            } catch (Throwable var4) {
                throw new UndeclaredThrowableException(var4);
            }
        }

        public final void sayHello() throws  {
            try {
                // 这里调用上边的DynamicProxy实例的 invoke方法。传入相应方法，
                //同时DynamicProxy中持有实际对象的引用，可以进行反射调用。
                super.h.invoke(this, m3, (Object[])null);
            } catch (RuntimeException | Error var2) {
                throw var2;
            } catch (Throwable var3) {
                throw new UndeclaredThrowableException(var3);
            }
        }

        public final String toString() throws  {
            try {
                return (String)super.h.invoke(this, m2, (Object[])null);
            } catch (RuntimeException | Error var2) {
                throw var2;
            } catch (Throwable var3) {
                throw new UndeclaredThrowableException(var3);
            }
        }

        public final int hashCode() throws  {
            try {
                return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
            } catch (RuntimeException | Error var2) {
                throw var2;
            } catch (Throwable var3) {
                throw new UndeclaredThrowableException(var3);
            }
        }

        static {
            try {
                m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[]{Class.forName("java.lang.Object")});
                m3 = Class.forName("com.acyouzi.jvm.DynamicProxyTest$IHello").getMethod("sayHello", new Class[0]);
                m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
                m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
            } catch (NoSuchMethodException var2) {
                throw new NoSuchMethodError(var2.getMessage());
            } catch (ClassNotFoundException var3) {
                throw new NoClassDefFoundError(var3.getMessage());
            }
        }
    }

### 反射，反射
分析下面代码的执行流程分析反射的过程：

    public class Demo {
        private int i;
        private int j;
        public String name;

        public Demo( String _name ) {
            this.name = _name;
        }

        private Demo() {
            this.name = "default";
        }

        public int add( int _i , int _j ){
            this.i = _i;
            this.j = _j;
            return this.i + this.j;
        }
    }
    public class TestReflect {
        public static void main(String[] args) throws Exception {
            Class c = Class.forName("com.acyouzi.reflect.Demo");
            Constructor con = c.getDeclaredConstructor();
            con.setAccessible(true);
            Demo d = (Demo)con.newInstance();
            Thread.sleep(30000);
            Method m = c.getMethod("add",int.class,int.class);

            for (int i = 0 ; i <=17 ;i++ ) {
                System.out.println(m.invoke(d, 3, 4));
            }
        }
    }

1. Class.forName 调用 native 方法 forName0 加载类。
2. getDeclaredConstructor() 检查权限，然后调用 getConstructor0 获取 Method 实例
3. getConstructor0 调用 privateGetDeclaredConstructors -> reflectionData 这里的 reflectionData 是个 SoftReference 因为是第一调用这个引用为空，所以需要调用 newReflectionData 新建一个引用。并更新 reflectionData 的值。因为可能由多个线程同时更新 reflectionData 所以这里使用乐观锁 casReflectionData 保证线程安全。
   privateGetDeclaredConstructors 中第一次调用reflectionData 获取的是一个新的对象，所以需要调用 native 方法 getDeclaredConstructors0 来获取有哪些构造方法并把方法保存在 reflectionData 中。

4. 然后返回到 getConstructor0 方法中，根据返回的一堆函数判断有没有需要的，如果有调用 getReflectionFactory().copyConstructor 函数最终是新建了一个 Constructor 对象，并把 root 指针指向找到的那个Constructor 对象。
   so. 不会返回 root Constructor。返回的都是包装过的。

5. setAccessible 设置的是 override 参数，这个参数并不是指语法层面理解的访问权限，而是指是否进行安全检查。
6. newInstance 创建对象实例，首先要调用acquireConstructorAccessor 因为是第一次调用没有缓存，需要创建一个新的 reflectionFactory.newConstructorAccessor 

        NativeConstructorAccessorImpl var3 = new NativeConstructorAccessorImpl(var1);
        DelegatingConstructorAccessorImpl var4 = new DelegatingConstructorAccessorImpl(var3);
        var3.setParent(var4);
        return var4;

   上面是创建 Accessor 的关键代码，创建了一个 NativeConstructorAccessor 一个委托的 DelegatingConstructorAccessor ，委托访问者会持有 NativeConstructorAccessor 的实例。

7. 获得 Accessor 之后通过 ca.newInstance(initargs); 创建实例，最后调用的是 NativeConstructorAccessorImpl 的 newInstance 实现
    
        public Object newInstance(Object[] var1) throws InstantiationException, IllegalArgumentException, InvocationTargetException {
            if(++this.numInvocations > ReflectionFactory.inflationThreshold() && !ReflectUtil.isVMAnonymousClass(this.c.getDeclaringClass())) {
                ConstructorAccessorImpl var2 = (ConstructorAccessorImpl)(new MethodAccessorGenerator()).generateConstructor(this.c.getDeclaringClass(), this.c.getParameterTypes(), this.c.getExceptionTypes(), this.c.getModifiers());
                this.parent.setDelegate(var2);
            }
            return newInstance0(this.c, var1);
        }

   上面的代码中有一个计数器 this.numInvocations 当他的值超过 Threshold (15) 时会创建一个新的 Accessor，这个 Accessor 是一个通过 asm 库直接拼装生成反射类。

   这是一种优化措施 NativeConstructorAccessor 速度慢，MethodAccessorGenerator 第一次生成时耗时较长。

### 自增陷阱

    public static void main(String[] args) {
        int count = 0;
        for (int i = 0; i < 10 ; i++) {
            count = count ++ ;
        }
        System.out.println(count);
    }

反编译之后

    public static void main(String[] args) {
        byte count = 0;
        for(int i = 0; i < 10; ++i) {
            int var3 = count + 1;
            count = count;
        }
        System.out.println(count);
    }

count = count ++ ; 这条语句正确与否与具体实现语言有关，c++中这条语句符合预期，php结果与java相同。

### 调用 javascript
    
    // test.js
    function test( str ){
        return str +" -- "+　name;
    }
    // java
    public static void main(String[] args) throws Exception {
        ScriptEngine engine = new ScriptEngineManager().getEngineByName("javascript");
        Bindings bind = engine.createBindings();
        // 上下文环境
        bind.put("name","acyouzi");
        engine.setBindings(bind, ScriptContext.ENGINE_SCOPE);
        engine.eval(new FileReader("test.js"));

        Invocable in = (Invocable)engine;
        String str =  (String)in.invokeFunction("test","abc");
        System.out.println(str);
    }

### 断言
断言默认情况下是关闭的，可以通过 参数-enableassertions或者-ea打开断言。

    assert <boolean 表达式> : ErrorMessage

断言一旦错误会抛出 AssertionError ，所以不要在对外公开的方法中使用，另外在生产环境不启用断言，所以不要在断言中带有业务逻辑。

### 取余运算

    public static void main(String[] args) {
        System.out.println(-1%2 == 1 ? "奇数": "偶数");
    }
    // 输出 偶数

取余运算大概可以使用下面的公式模拟 (int d1 , int d2) -> d1 - d1 / d2 * d2

所以 -1 % 2 的结果是 -1 而不是 1 ，so . 判断奇偶用 i%2 == 0 ? "偶数":"奇数"

### 整形池

    public static void main(String[] args) {
        int num = 24;
        Integer i = new Integer(num);
        Integer j = new Integer(num);
        System.out.println(i == j);

        i = num;
        j = num;
        System.out.println(i == j);

        i = Integer.valueOf(num);
        j = Integer.valueOf(num);
        System.out.println(i == j);
    }
    // 输出
    false
    true
    true

先来看一下 valueOf 的代码
    
    public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
    
其中 IntegerCache.low 等于 -128 , IntegerCache.high 等于 "java.lang.Integer.IntegerCache.high" 设置的值或者等于默认值 127。在 low 和 high 之间的值都是通过常量池获取。

然后再来看这两句 

    i = num;
    j = num;

java 自动装箱的语法糖把这两句装换为了 

    i = Integer.valueOf(num);
    j = Integer.valueOf(num);

所以，这里给的警示是 加锁最好别往 Integer 对象上加，String 也一样。

### 构造代码块
会插入到每个构造函数的 super() 语句之后，如果构造函数中通过 this() 调用了其他构造函数则不会插入构造代码块，这样保证了构造代码块只会执行一次。

在使用匿名类时，没法显示写出构造函数，只能通过构造代码块实现。

### 静态内部类和普通内部类的区别
1. 普通内部类可以直接访问外部类的属性、方法，即使是private,因为他持有一个外部类的引用。静态内部类只能访问外部类的静态资源(包括 private).
2. 普通内部类实例不能脱离外部类实例，在回收时会一起被回收。静态内部类可以独立存在。
3. 普通内部类不能声明 static 方法和变量

### equals 与 hashCode
重写 equals 方法后一定要重写 hashCode 方法

    public boolean equals(Object obj) {
        if( obj != null && obj.getClass() == this.getClass()){
            // 自己的判断逻辑
        }
        return false;
    }
    public int hashCode() {
        // org.apache.commons.lang
        return new HashCodeBuilder().append("some attr").toHashCode();
    }

### String 的 intern()

    public static void main(String[] args) {
        String str1 = "test1";
        String str2 = new String("test1");
        String str3 = str2.intern();

        System.out.println(str1 == str2 );
        System.out.println(str1 == str3 );
        System.out.println(str2 == str3 );
    }
    // 输出结果
    false
    true
    false

直接赋值是把字符串放在常量池中，intern 方法是 native 方法，会检查常量池中有没有相同的字符串。有直接返回，没有把字符串加入再返回。

String 是一个不可变对象，他提供的方法如果有 String 返回值就会创建一个新的对象，不对原来的对象进行修改。

### String StringBuffer StringBuilder
1. String 不可变
2. StringBuffer 可变，线程安全
3. StringBuilder 可变，非线程安全

如果需要拼接字符串，使用 StringBuffer StringBuilder 效率要远远高于使用 + 或者 concat()
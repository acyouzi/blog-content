---
title: java 杂记
date: 2016-11-16 16:38:50
author: "acyouzi"
# cdn: header-off
# header-img: img/dog.jpg
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

   上面的代码中有一个计数器 this.numInvocations 当他的值超过 Threshold (15) 时会创建一个新的 Accessor，这个 Accessor 是一个通过 asm 库直接拼装生成反射字节码的类。

   这是一种优化措施 NativeConstructorAccessor 速度慢，MethodAccessorGenerator 第一次生成是耗时较长。

#### 
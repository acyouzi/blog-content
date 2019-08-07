---
title: spring 基本使用
date: 2017-01-15 20:07:00
author: "acyouzi"
description: 
tags:
	- java
---

### DI 依赖注入
请看下面的例子

#### Step.1
首先定义 Animal 接口

    public interface Animal {
        public String sayHello();
    }

然后定义一个狗的实现类，并且添加 @Component("dog") 注解配置为一个 Bean 

    @Component("dog")
    public class Dog implements Animal {
        public String sayHello() {
            return "wang wang wang !!!";
        }
    }

然后定义一个人，他养了一个动物

    public class People {
        private Animal animal;
        public People(Animal animal) {
            this.animal = animal;
        }
        public void playWithAnimal(){
            System.out.println(animal.sayHello());
        }
    }

下面是一个配置文件，用于配置开启注解自动扫描，同时，我还想不使用 @Component 注解把 People 也配置为一个 Bean 可以直接配置在这个配置文件中

    @Configuration
    @ComponentScan(basePackageClasses = {AopSpringConfig.class, DiSpringConfig.class})
    public class BaseConfiguration {
        @Bean(name = "people")
        public People getPeople(Animal animal){
            return new People( animal );
        }
    }

@Configuration 说明这个类可能通过 @Bean 注解声明了一些 Bean,同时还有可能进行了一些其他配置
@ComponentScan(basePackageClasses = {DiSpringConfig.class}) 这个标签用于开启自动扫描注解，配置 basePackage 有多种配置方法，推荐配置 basePackageClasses 指向包下的一个空接口，这样在重构的时候比不容易出问题
@Bean 作用跟 @Component 一样，只不过用的地方不同
public People getPeople(Animal animal) 方法需要的参数会自动注入进去

下面编写测试类

    @RunWith(SpringJUnit4ClassRunner.class)
    @ContextConfiguration(classes = {BaseConfiguration.class})
    public class SpringTest {
        @Autowired
        private People people;
        @Test
        public void testDi(){
            people.playWithAnimal();
        }
    }

需要依赖 spring-test 包
@RunWith 是用来搞定环境的，必须要写
@ContextConfiguration(classes = {BaseConfiguration.class}) 指定配置文件
@Autowired 表明需要注入 People 对象

测试发现装配成功，方法正常执行。

#### Step.2
在上面的基础上再编写 cat 类

    @Component("cat")
    public class Cat implements Animal {
        public String sayHello() {
            return "miao miao miao !!!";
        }
    }

然后重写运行测试，出错了，哈哈哈，如何解决？
这里问题是自动装配存在歧义，可以使用 @Primary 或者 @Qualifier() 来消除歧义

1. 在 Dog 类上添加 @Primary 注解，再次运行发现输出 wang wang wang !!!
2. 在 Dog 类上添加 @Qualifier("wang") 在 Cat 类上添加 @Qualifier("miao") 然后再对 People 略作修改，不再通过 java配置方式声明为 Bean 而是通过 @Component 方式

        @Component
        public class People {
            @Autowired
            @Qualifier("cat")
            private Animal animal;
            
            public void playWithAnimal(){
                System.out.println(animal.sayHello());
            }
        } 

    再次运行测试，歧义消除，@Qualifier 创建限定符可以配合 @Component @Bean 使用，但是注入时好像只能在 @Autowired 上起作用

#### Step.3
现在，假设在开发的时候我们需要人养的宠物是狗，测试的时候人养的宠物是猫，应该怎么做？ 用 @Profile

给 Dog 加上 @Profile("dev") 给 Cat 加上 @Profile("qa") 

    @Component("dog")
    @Profile("dev")
    public class Dog implements Animal {
        public String sayHello() {
            return "wang wang wang !!!";
        }
    }

    @Component("cat")
    @Profile("qa")
    public class Cat implements Animal {
        public String sayHello() {
            return "miao miao miao !!!";
        }
    }

至于具体是 dev 还是 qa 依赖于 spring.profile.active spring.profile.default 两个属性
这两个属性可以通过 jvm 参数，环境变量, @ActiveProfiles 注解等方式配置，这里我们在 SpringTest类上添加注解 @ActiveProfiles("dev") 把环境配置为 dev 测试，运行测试输出 wang wang wang !!!

#### Step.4
现在我的系统中有一个属性叫做 animal.type 假设我希望如果这个属性值等于 cat 则调用 cat 否则调用 dog 该怎么做？
使用 @Conditional() 注解可以完成上面的要求 @Conditional() 注解需要一个实现了 condition 接口的类作为判断逻辑

    public class DogConditionImpl implements Condition {
        public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
            Environment env = context.getEnvironment();
            return env.containsProperty("animal.type")&&env.getProperty("animal.type").equals("Dog");
        }
    }

    @Component("dog")
    @Conditional(DogConditionImpl.class)
    public class Dog implements Animal {
        public String sayHello() {
            return "wang wang wang !!!";
        }
    }

    public class CatConditionImpl implements Condition {
        public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
            Environment env = context.getEnvironment();
            return env.containsProperty("animal.type")&&env.getProperty("animal.type").equals("cat");
        }
    }
    @Component("cat")
    @Conditional(CatConditionImpl.class)
    public class Cat implements Animal {
        public String sayHello() {
            return "miao miao miao !!!";
        }
    }

在运行 java 测试的时候，启动参数添加 -Danimal.type=cat 然后运行测试类，输出 miao miao miao !!!

#### Step.5
假设现在这个人养了两只猫，然后两只猫每次叫都能记录自己叫了几次，代码如下
    
    @Component("cat")
    public class Cat implements Animal {
        public int i = 0;
        public String sayHello() {
            ++i;
            return "miao : " +i;
        }
    }

    @Component
    public class People {
        @Autowired
        private Animal cat1;
        @Autowired
        private Animal cat2;

        public void playWithAnimal(){
            System.out.println(cat1.sayHello());
            System.out.println(cat2.sayHello());
        }
    }

然后测试发现运行结果为
    
    miao : 1
    miao : 2

原来spring 注入的两个对象，是用一只猫，这里就要涉及到spring 的作用域了，spring 定义了四个作用域：

    * 单例(ConfigurableBeanFactory.SCOPE_SINGLETON) 整个应用只有一个实例
    * 原型(ConfigurableBeanFactory.SCOPE_PROTOTYPE) 每次注入都是创建一个新的实例
    * Session 用于web 一个会话一个
    * Request 用于web 一个请求一个

作用域可以通过 @Scope() 来修改，我们可以把猫类的作用域注解为原型 @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE) 来解决这个问题

#### Step.6 
目前为止小猫小狗还有人的使命已经结束了。下面来看一看运行时值的注入

1. 创建 app.properties 随便输入点内容
    com.acyouzi.spring=test
    lol=happy
2. 使用 @PropertySource("app.properties") 引入 app.properties
        @Configuration
        @ComponentScan(basePackageClasses = {AopSpringConfig.class, DiSpringConfig.class})
        @PropertySource("app.properties")
        public class BaseConfiguration {
        }
3. 使用 Environment 变量访问属性值
        @RunWith(SpringJUnit4ClassRunner.class)
        @ContextConfiguration(classes = {BaseConfiguration.class})
        public class SpringTest {
            @Autowired
            Environment env;
            @Test
            public void simpleReadPropertyTest(){
                System.out.println("com.acyouzi.spring = " + env.getProperty("com.acyouzi.spring"));
                System.out.println("lol = " + env.getProperty("lol"));
            }
        }

    注意需要注入 Environment
4. 使用 @Value + 占位符，占位符的形式是 "${...}", 注意两个 {} 中间不能有空格

        @Component
        public class ReadProperty {
            @Value("${com.acyouzi.spring}")
            private String spring;
            private String lol;

            public ReadProperty( @Value("${lol}") String lol) {
                this.lol = lol;
            }
            @Override
            public String toString() {
                return "ReadProperty{" +
                        "spring='" + spring + '\'' +
                        ", lol='" + lol + '\'' +
                        '}';
            }
        }

        输出 ReadProperty{spring='test', lol='happy'}

    @Value 可以用在属性上，也可以用在函数上
    @Value 内部可以直接写普通字符串，也可以写占位符，SpEL 表达式

### SpEL 表达式
SpEL 的形式是 "#{...}" ,表达式能够使用 bean 的 id 来引用 bean, 调用方法，访问对象属性，算数逻辑关系运算，正则表达式，集合操作
    
    "#{ systemProperties['com.acyouzi.spring'] }" 
    "#{ 'aaabbccdd' marches 'b+c' }"
    "#{ T(java.lang.Math).random() }"
    "#{ readProperty.toString() }"

总之 SpEL 很强大，spel 中还有判断空值的类型安全运算符( ?. ), 集合索引运算, 过滤集合的查询运算 xxx.?[],xxx.^[],xxx.![] ,但是 SpEL 在程序里缺乏可见性，不容易调试，所以最好不要写太复杂的。

### AOP 
Spring Aop 是在运行时织入的，基于动态代理实现的，所以 Spring Aop 只支持方法连接点, Spring Aop 有如下5种类型的通知

1. 前置通知
2. 后置通知
3. 返回通知
4. 异常通知
5. 环绕通知

Spring 借助 AspectJ 的切点表达式来语言来定义 Spring 切面，但是 Spring 仅支持部分切点表达式,但是实际上貌似我一般只使用 execution() 指示器,同时 spring 还引入了一个新的 bean() 指示器，还有下面几个指示器，我也不太会用

* within：用于匹配指定类型内的方法执行；
* this：用于匹配当前AOP代理对象类型的执行方法；注意是AOP代理对象的类型匹配，这样就可能包括引入接口也类型匹配；
* target：用于匹配当前目标对象类型的执行方法；注意是目标对象的类型匹配，这样就不包括引入接口也类型匹配；
* args：用于匹配当前执行的方法传入的参数为指定类型的执行方法；
* @within：用于匹配所以持有指定注解类型内的方法；
* @target：用于匹配当前目标对象类型的执行方法，其中目标对象持有指定的注解；
* @args：用于匹配当前执行的方法传入的参数持有指定注解的执行；
* @annotation：用于匹配当前执行方法持有指定注解的方法；


下面简单介绍切点的编写
    
    execution( modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern (param-pattern) throws-pattern? )

上面就是切点表达式的格式，带 ? 的代表可以省略。下面举几个例子

1. 匹配任意访问修饰符，任意返回值的 com.acyouzi.spring.aop.Admin 类的 任意数量类型参数的 managementSystem 方法
        @Before("execution(** com.acyouzi.spring.aop.Admin.managementSystem(..))") 
        @Before("execution(* com.acyouzi.spring.aop.Admin.managementSystem(..))")

2. 匹配以 set 开头的任意方法
        @Before("execution(* set(..))")

3. 匹配 com.acyouzi.spring.aop 包下所有类的所有方法
        @Before("execution(* com.acyouzi.spring.aop.*.*(..))") 

4. 匹配 com.acyouzi.spring 包及其子包下所有类的所有方法
        @Before("execution(* com.acyouzi.spring..*.*(..))") 

5. 匹配 com.acyouzi.spring.aop.Admin 类下的 managementSystem 方法，但是只有 bean 的 ID 为 admin
        @Before("execution(* com.acyouzi.spring.aop.Admin.managementSystem(..)) and bean('admin') ") 

6. 匹配 com.acyouzi.spring.aop.Admin 类下的 managementSystem 方法，但是只有 bean 的 ID 不为 admin
        @Before("execution(* com.acyouzi.spring.aop.Admin.managementSystem(..)) and !bean('admin') ") 

下面是一个例子
#### step.1
现在有一个管理员，他可以管理系统，但是管理员只能管理 ID 比他大的，并且不能是特权ID 110 的账户：
    
    @Component()
    public class AdminAction {
        public int myid = 10;
        public boolean managementSystem(int id) throws Exception {
            if( id == 110 ){
                throw new Exception(" id not found");
            }
            return id > myid ;
        }
    }

现在希望能记录管理员的每一次操作，并且对成功或者失败的结果进行记录，可以写一个切面,并注册为 Bean：
    
    @Component
    @Aspect
    public class LogService {
        @Before("execution(* com.acyouzi.spring.aop.AdminAction.managementSystem(..))")
        public void haveVisit(){
            System.out.println("有一条访问");
        }

        @AfterReturning("execution(* com.acyouzi.spring.aop.AdminAction.managementSystem(..))")
        public void normalState(){
            System.out.println("访问完成");
        }

        @AfterThrowing("execution(* com.acyouzi.spring.aop.AdminAction.managementSystem(..))")
        public void exceptionState(){
            System.out.println("访问出现问题");
        }
    }

    // 配置类上添加注解
    @EnableAspectJAutoProxy
    public class BaseConfiguration {}

@Aspect 注解表示这是一个切面
@Before 注解表示在切点表达式匹配的方法执行前执行服务方法。
@AfterReturning 方法正常结束后调用
@AfterThrowing 方法抛出异常时调用
还有一个 @After 方法，在正常或者异常结束方法时都会调用

这几个注解都是 AspectJ 的注解，所以需要引入 AspectJ 依赖 

    compile group: 'org.aspectj', name: 'aspectjrt', version: '1.8.9'
    compile group: 'org.aspectj', name: 'aspectjweaver', version: '1.8.9'

下面是测类

    @RunWith(SpringJUnit4ClassRunner.class)
    @ContextConfiguration(classes = {BaseConfiguration.class})
    public class SpringTest {
        @Autowired
        AdminAction admin;
        @Test
        public void AopTest() throws Exception {
            admin.managementSystem(210);
        }
    }

可以运行测试，看到日志输出

#### step.2
上面的切面，相同的表达式我们重复了三遍，这个对后面代码重构，表达式维护不太友好，我们可以通过 @Pointcut 注解定义一个可重用的切入点

    @Component
    @Aspect
    public class LogService {
        @Pointcut("execution(* com.acyouzi.spring.aop.AdminAction.managementSystem(..)) and !bean('adminAction')")
        public void managementSystemPointCut(){}
        @Before("managementSystemPointCut()")
        public void haveVisit(){
            System.out.println("有一条访问");
        }

        @AfterReturning("managementSystemPointCut()")
        public void normalState(){
            System.out.println("访问完成");
        }

        @AfterThrowing("managementSystemPointCut()")
        public void exceptionState(){
            System.out.println("访问出现问题");
        }
    }

#### step.3 
上面这些切面方法，完全可以写在一个方法里面，而且更方便进行控制，可以使用环绕注解 @Around 实现
    
    @Aspect
    public class LogService {
        @Pointcut("execution(* com.acyouzi.spring.aop.AdminAction.managementSystem(..))")
        public void managementSystemPointCut(){}

        @Around("managementSystemPointCut()")
        public boolean watchManagement(ProceedingJoinPoint joinPoint) throws Throwable {
            System.out.println("一次访问:");
            try {
                boolean res = (Boolean)joinPoint.proceed();
                System.out.println("正常结束");
                return res;
            } catch (Throwable throwable) {
                System.out.println("发生一次");
                throw throwable;
            }
        }
    }

这里有几点需要注意 
@Around 注解的方法 需要有一个 ProceedingJoinPoint joinPoint 参数，这个参数用来调用被调用的对象。
这个方法返回值必须与表达式匹配的实际方法的返回值一致
joinPoint.proceed() 会最终调用实际方法，返回值是实际方法的返回值，但是类型是 Object 类型，需要我们自己转换。

实际上，在 @Around 注解的方法中可以完成错误重试，安全验证等多种任务

#### step.4 
现在新的需求是希望能够记录每次请求的参数值，并且每次请求参数值，以及方法返回参数值，可以写成下面的形式

    @Aspect
    public class LogService {
        @Pointcut("execution(* com.acyouzi.spring.aop.AdminAction.managementSystem(int)) && args(id)")
        public void managementSystemPointCut(int id){}

        @Before("managementSystemPointCut(id)")
        public void haveVisit(int id){
            System.out.println("有一条访问 : "+id);
        }

        @Around("managementSystemPointCut(id)")
        public boolean watchManagement(ProceedingJoinPoint joinPoint,int id) throws Throwable {
            System.out.println("一次访问 : "+id);
            try {
                boolean res = (Boolean)joinPoint.proceed(new Object[]{id});
                System.out.println("正常结束 : " +res);
                return res;
            } catch (Throwable throwable) {
                System.out.println("发生异常");
                throw throwable;
            }
        }
    }

注意切点表达式 execution(* com.acyouzi.spring.aop.AdminAction.managementSystem(int)) && args(id) 多个一个 args 这个的值要与 PointCut 还有 各种通知方法的参数一致。

另外其实 Before After AfterReturning AfterThrowing，几个事件都可以注入 JoinPoint 对象，完全可以通过这个对象获得方法参数，而不用去修改切点表达式，我觉得使用 JoinPoint 获取输入参数要好过使用上面的这种方法 

#### other
Before AfterReturning AfterThrowing里面可以注入 JoinPoint，注意 JoinPoint 相比 @Around 注入的 ProceedingJoinPoint 少了 proceed 方法。 
@AfterReturning 可以注入结果
@AfterThrowing 可以注入异常值
    
    @Before("managementSystemPointCut(id)")
    public void haveVisit(JoinPoint jp , int id){
        System.out.println(jp.getArgs()[0]);
        System.out.println("有一条访问 : "+id);
    } 
    @AfterReturning(value = "execution(* com.acyouzi.spring.aop.AdminAction.managementSystem(..))", returning = "res")
    public void haveVisit( boolean res ){
        System.out.println("res : "+res);
    }
    @AfterThrowing(value = "execution(* com.acyouzi.spring.aop.AdminAction.managementSystem(..))",  throwing = "res")
    public void throwVisit( Exception res ){
        System.out.println("res : "+res);
    }


至于,配置 xml 使用 spring 的方式...让xml去死吧
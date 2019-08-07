---
title: spring 数据库集成以及缓存
date: 2017-01-17 14:47:19
author: "acyouzi"
description: 
tags:
	- java
---

### Spring Jdbc
Spring 提供的持久化相关的异常都是非捕获异常，可以不用捕获，这与其设计思想有关。spring 认为触发异常的很多问题是无法在 catch 中解决的，所以不应该强制开发人员去捕获异常。

首先添加几个依赖

    compile group: 'org.apache.commons', name: 'commons-dbcp2', version: '2.1.1'
    compile group: 'org.springframework', name: 'spring-jdbc', version: '4.3.5.RELEASE'
    compile group: 'mysql', name: 'mysql-connector-java', version: '6.0.5'

#### DBCP 连接池配置
1. 创建配置文件 jdbc.properties 写入配置

    db.driver=com.mysql.cj.jdbc.Driver
    db.jdbcurl=jdbc:mysql://192.168.192.132/spring_test?characterEncoding=utf8
    db.username=root
    db.password=

2. 配置 Bean
    
        @Configuration
        public class JdbcSpringConfig {
            @Value("${db.driver}")
            private String driver;
            @Value("${db.jdbcurl}")
            private String jdbcurl;
            @Value("${db.username}")
            private String username;
            @Value("${db.password}")
            private String password;
            @Bean
            public DataSource getDBCPDataSource(){
                System.out.println(jdbcurl + username + password );
                BasicDataSource dataSource = new BasicDataSource();
                dataSource.setDriverClassName(driver);
                dataSource.setUrl(jdbcurl);
                dataSource.setUsername(username);
                dataSource.setPassword(password);
        //        dataSource.setInitialSize(5);
        //        dataSource.setMinIdle(5);
        //        dataSource.setMaxIdle(10);
        //        dataSource.setMaxTotal(-1);
                return dataSource;
            }
        }

    这里特别说明一下使用新版本 mysql 驱动容易出现的 serverTimezone 错误：
    The server time zone value 'ÖÐ¹ú±ê×¼Ê±¼ä' is unrecognized or represents more than one time zone. You must configure either the server or JDBC driver (via the serverTimezone configuration property) to use a more specifc time zone value if you want to utilize time zone support.

    这个错误是因为 mysql 在启动时会读取系统时区，但是不知道为啥在 windows 下边读取到的是乱码
        
        show variables like "%time_zone%";
        +------------------+-------------------+
        | Variable_name    | Value             |
        +------------------+-------------------+
        | system_time_zone | ???ú±ê×??±?? |
        | time_zone        | SYSTEM            |
        +------------------+-------------------+
        2 rows in set (0.00 sec)

    解决方法有：
    
    可以在 db.jdbcurl 中添加 serverTimezone=UTC

        db.jdbcurl=jdbc:mysql://192.168.192.132/spring_test?serverTimezone=UTC&characterEncoding=utf8
    
    也可以想办法在服务器端修改, 当然最好还是找个 linux mysql 服务器, 把服务器时区改成 CST

        show variables like "%time_zone%";
        +------------------+--------+
        | Variable_name    | Value  |
        +------------------+--------+
        | system_time_zone | CST    |
        | time_zone        | SYSTEM |
        +------------------+--------+
        2 rows in set (0.00 sec)
 
#### 使用 Spring 数据源连接
spring-jdbc 包中提供了三个数据源

    DriverManagerDataSource 每个连接请求都返回一个新的连接
    SimpleDriverDataSource 类似 DriverManagerDataSource
    SingleConnectionDataSource 每个请求返回同一个连接

可以使用使用如下代码配置
    @Bean
    public DataSource getSpringDataSources(){
        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setDriverClassName(driver);
        ds.setUrl(jdbcurl);
        ds.setUsername(username);
        ds.setPassword(password);
        return ds;
    }

#### 多数据源切换
可以使用 @Profile 注解来实现在 开发 测试 线上 不同环境的数据源自动切换

### JdbcTemplate
因为在数据库访问中存在大量重复的模板代码，这些代码是对工作能力的浪费。Spring 对这些代码进行了比较好的封装，Spring 提供了两个模板类供我们选择。

    JdbcTemplate 基本模板
    NamedParameterJdbcTemplate 执行查询时可以将值以命名参数的形式绑定到 SQL 中，而不是简单的使用索引参数。

1. 配置 Bean

        @Bean
        public JdbcTemplate jdbcTemplate(DataSource dataSource){
            return new JdbcTemplate(dataSource);
        }

2. 编写测试类

        @RunWith(SpringJUnit4ClassRunner.class)
        @ContextConfiguration(classes = {BaseConfiguration.class})
        public class SpringTest {
            @Autowired
            private JdbcOperations operations;
            @Test
            public void JdbcTest() throws Exception {
                // insert
                int res = operations.update("INSERT INTO `test`(`name`, `passwd`) VALUES (?,?)", "test", "test");
                System.out.println(res);
                // select
                List<User> users = operations.query("select * from test where id=?", (rs, rowNum) -> {
                    return new User(rs.getInt("id"), rs.getString("name"), rs.getString("passwd"));
                }, 1);
                System.out.println(users);
            }
        }

3. 命名参数使用

        Map params = new HashMap();
        params.put("name","xxxx");
        params.put("passwd","xxxx");
        namedTemplate.update("INSERT INTO `test`(`name`, `passwd`) VALUES (:name,:passwd)",params);

4. JPA
    spring 还能与 JPA 模块便捷集成，提供了应用程序管理类型和容器管理类型两种配置方式，一般都会配置成容器管理类型。至于具体配置这里就不做过多介绍了，配置中需要提供一个 JPAVendorAdapter,Spring 提供了 EclipseLinkJPAVendorAdapter,HibernateJPAVendorAdapter,OpenJPAVendorAdapter 等多个 JPA 厂商适配器

### Spring redis
Spring 还提供了跟各种非关系型数据简单集成的工具，能够集成 MongoDB,Neo4j,Redis 等非关系型数据库。同时提供了一系列的注解以及模板方法来简化 dao 层的开发。下面简答介绍 Redis 相关的内容.

spring 为四种 redis 客户端实现了连接工厂,分别是 

    JedisConnectionFactory
    JredisConnection
    LettuceConnectionFactory
    SrpConnectionFactory

spring 提供了 RedisTemplate StringRedisTemplate 连个模板类，对重复步骤进行了封装。在 RedisTemplate 中可以如上面代码显示的一样设置键和值的序列化方法

可用的序列化方法有 
    
    GenericToStringSerializer 
    Jackson2JsonRedisSerializer 
    JdkSerializationRedisSerializer 
    OxmSerializer 
    StringRedisSerializer 

1. 注册 Bean

        @Configuration
        public class RedisConfiguration {
            @Bean
            public RedisConnectionFactory getRedisConnectionFactory(){
                JedisConnectionFactory factory = new JedisConnectionFactory();
                factory.setHostName("192.168.192.128");
                factory.setPort(6379);
                return  factory;
            }
            @Bean
            public RedisTemplate<String,String> getRedisTemplate(RedisConnectionFactory factory){
                RedisTemplate<String,String> template = new RedisTemplate<String, String>();
                template.setConnectionFactory(factory);
                template.setKeySerializer(new StringRedisSerializer());
                template.setValueSerializer(new StringRedisSerializer());
                return template;
            }
        }

2. 编写测试方法

        @RunWith(SpringJUnit4ClassRunner.class)
        @ContextConfiguration(classes = {BaseConfiguration.class})
        public class SpringTest {
            @Autowired
            private RedisTemplate<String,String> redisTemplate;
            @Test
            public void redisTest() throws Exception {
                // 简单值操作
                redisTemplate.opsForValue().set("test","test");
                System.out.println(redisTemplate.opsForValue().get("test"));

                // list
                redisTemplate.opsForList().leftPush("list:test","acyouzi");
                System.out.println(redisTemplate.opsForList().rightPop("list:test"));

                // set
                redisTemplate.opsForSet().add("set:test","aaa","bbb","ccc");
                redisTemplate.opsForSet().remove("set:test","aaa");
                System.out.println(redisTemplate.opsForSet().size("set:test"));
            }
        }

### 缓存
Spring 对缓存功能提供了声明式的支持，并内置了几个缓存管理器

    SimpleCacheManager 
    NoOpCacheManager 
    CompositeCacheManager // 组合使用多个缓存
    ConcurrentMapCacheManager // 线程安全的
    JCacheCacheManager 
    GuavaCacheManager 

Spring Data Redis 中还提供了一个 

    RedisCacheManager

 下面是一个配置 Redis 作为 CacheManager 的示例。

    @Configuration
    @EnableCaching
    public class CacheConfig {
        @Bean
        public CacheManager cacheManager(RedisTemplate<String ,String> redisTemplate){
            return new RedisCacheManager(redisTemplate);
        }
    }

首先是要 @EnableCaching，然后把 CacheManager 注册为 Bean. RedisCacheManager 需要一个 RedisTemplate 就是我们前边介绍 redis 时注册的 redisTemplate  

#### 缓存相关注解
1. @Cacheable
    方法调用之前首先在缓存中检测方法的返回值，如果能够找到，就返回缓存中的值。否则调用方法，并把方法返回值放到缓存中。一般需要反复调用的查询方法，而且值不经常改变，就可以添加该方法。

2. @CachePut
    方法调用前不检查缓存，方法始终会被调用，方法返回值会被放到缓存中。如果一个方法是去向数据库中添加或者修改某个值，而这个值又是在其他地方频繁查询的，那么这个方法上应该添加该注解

3. @CacheEvict
    在缓存中清除一个或多个条目。如果一个方法的任务是删除一个数据库中的值，而这个值刚好在缓存中也有保存，那么应该使用这个注解删除缓存中相关的值。

为了方便可以直接把注解加载需要缓存的方法的接口上.这样每个实现接口的方法都能应用缓存的配置，也可应放到类上，这样类的每个方法都会被缓存。

@Cacheable 与 @CachePut 中有一些共有的参数可以用于对缓存做一些控制。

    value 缓存名 string[]
    condition 条件 SpEL 表达式 如果值为 false 则不放入缓存
    unless 条件 SpEL 表达式 如果值为 true 则不放入缓存， 如 #result == null
    key 缓存的键 SpEL 表达式，默认好像是所有的传入参数

@CacheEvict 的参数有

    value 
    key 
    condition
    allEntries boolean 如果为 true 删除所有条目
    beforeInvocation boolean 如果为 true 在方法调用之前删除，默认为 false

并且提供了多个用于定义缓存规则的 SpEL 

    #root.methodName 缓存方法的名字 
    #root.method 缓存方法
    #root.target 目标对象
    #root.targetClass 目标对象的类
    #root.args[0] 传递给缓存方法的参数，形式为数组
    #root.caches[0] 方法执行时所对应的缓存， 形式为数组
    #argument 任意方法参数名
    #result 方法调用的返回值



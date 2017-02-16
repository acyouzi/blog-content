---
title: Maven 配置使用以及常用插件总结
date: 2017-02-17 01:40:07
author: "acyouzi"
tags:
	- maven
---
# 概述
一直在用 maven 学的东西零零散散的，不成系统。正好抽空把 maven 我所了解的东西整理了一下，备忘，也方便以后查找。

## 简单配置
1. apache-maven-3.3.9/conf/settings.xml 修改本地仓库位置

        <localRepository>E:/apache-maven-3.3.9/localRepository</localRepository>
        
2. 新建 localRepository 目录，在目录内拷贝一份 conf/settings.xml. 
        
3. 中央仓库定义在 lib/maven-model-builder-3.3.9.jar/org/apache/maven/model/pom.xml 

        <repositories>
            <repository>
                <id>central</id>
                <name>Central Repository</name>
                <url>https://repo.maven.apache.org/maven2</url>
                <layout>default</layout>
                <snapshots>
                    <enabled>false</enabled>
                </snapshots>
            </repository>
        </repositories>
        
        <pluginRepositories>
            <pluginRepository>
                <id>central</id>
                <name>Central Repository</name>
                <url>https://repo.maven.apache.org/maven2</url>
                <layout>default</layout>
                <snapshots>
                <enabled>false</enabled>
                </snapshots>
                <releases>
                    <updatePolicy>never</updatePolicy>
                </releases>
            </pluginRepository>
        </pluginRepositories>

4. 配置 idea
    
        file -> other settings -> default settings -> 
        Build,Execution,Deployment -> build tools -> maven

        修改 maven 指向我们自己的 maven 程序，不要使用默认的。
        修改 user settings file 指向 第三步拷贝的 localRepository/settings.xml 


## 设置使用的仓库
在 apache-maven-3.3.9/localRepository/settings.xml 中设置，profile 配置工厂，activeProfiles 激活配置。 这里使用 aliyun 的工厂。当 localRepository 不存在会去 Aliyun Repository 找，如果 Aliyun Repository 挂掉了，会去默认的 Central 工厂找。

    <profiles>
        <profile>
          <id>AliyunRepository</id>
        
          <repositories>
            <repository>
                <id>aliyun</id>
                <name>Aliyun Repository</name>
                <url>http://maven.aliyun.com/nexus/content/repositories/central</url>
                <layout>default</layout>
                <releases>
                    <enabled>true</enabled>
                    <updatePolicy>never</updatePolicy>
                </releases>
                <snapshots>
                    <enabled>false</enabled>
                </snapshots>
            </repository>
          </repositories>
          
          <pluginRepositories>
            <pluginRepository>
              <id>aliyun</id>
              <name>Aliyun Repository</name>
              <url>http://maven.aliyun.com/nexus/content/repositories/central</url>
              <layout>default</layout>
              <snapshots>
            	<enabled>false</enabled>
              </snapshots>
              <releases>
            	<updatePolicy>never</updatePolicy>
              </releases>
            </pluginRepository>
          </pluginRepositories>
        </profile>
    </profiles>
    <activeProfiles>
        <!--  激活 prifile -->
        <activeProfile>AliyunRepository</activeProfile>
    </activeProfiles>

或者可以配置 mirrors .这里我们使用的 central 或者其他配置的工厂都会映射到我们配置的 url. 如果这个 url 挂掉了，那就是真的找不到了。

    <mirrors>
        <mirror>
            <id>testMirror</id>
            <!--repositoryId-->
            <mirrorOf>*</mirrorOf>
            <name>All of the repository</name>
            <url>http://maven.aliyun.com/nexus/content/repositories/central</url>
        </mirror>
    </mirrors>

## deply jar 到 nexus 的配置
首先需要配置仓库, pom.xml 中

    <distributionManagement>
        <repository>
            <id>Releases</id>
            <url>xxxx</url>
        </repository>
        <repository>
            <id>Snapshots</id>
            <url>xxxx</url>
        </repository>
    </distributionManagement>

然后需要设置 server 的用户名密码，settings.xml

    <servers>
        <server>
          <!--id 要跟上面 repository 的相同-->
          <id>Releases</id>
          <username>repouser</username>
          <password>repopwd</password>
        </server>
    </servers>

## 依赖范围
    
    <scope>test</scope>
    
    compile
    
        编译范围有效，编译和打包时会把依赖存储进去
    
    provided
    
        在编译和测试阶段有效，打包不会依赖，比如 servlet-api , 在 web 服务器上 servlet-api 已经存在了，如果再打包会引起冲突。
    
    runtime
    
        在运行时依赖，在编译的时候不依赖，比如 mysql 的驱动包，在整个项目代码中没有直接使用，但是在运行时会加载这个类。
    
    test
    
        测试范围有效，编译/打包时不会使用这个依赖
    
    system
        
        忘记了，没用到过


## 依赖冲突

    a -> xx.1
    b -> xx.2
    
    c -> a,b  
    c 依赖 xx.1 还是 xx.2 呢？

    a,b 和 xx 的依赖是直接依赖
    c 和 xx 的依赖是间接依赖
    
    在 c 中如果先声明的依赖是 a 则, 依赖 xx.1，也就是间接依赖级别相同，依赖先声明的那个。
    
    当依赖级别不相同时，选择依赖层次最短的那一个。

## 排除依赖
如果 xxxx 依赖一个 junit 3.xx 但是我希望他不要使用他自己的 junit 依赖，可以使用 exclusions 排除依赖

    <dependency>
        <groupId>xxxx</groupId>
        <artifactId>xxxx</artifactId>
        <version>4.12</version>
        <exclusions>
            <exclusion>
                <groupId>junit</groupId>
                <artifactId>junit</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

## maven 隐式变量

    ${basedir} 项目根目录
    ${project.build.directory} 构建目录，缺省为target
    ${project.build.outputDirectory} 构建过程输出目录，缺省为target/classes
    ${project.build.finalName} 产出物名称，缺省为${project.artifactId}-${project.version}
    ${project.packaging} 打包类型，缺省为jar
    ${project.xxx} 当前pom文件的任意节点的内容

## maven 本地编写的模块被其他模块引用

    执行 mvn install 安装到本地仓库
    
## maven 聚合，一次管理多个模块t
在 idea 中新建一个 maven 作为父项目，然后可以按照下面的方法建立和管理子项目。

    file -> new -> Model... 
    在一个项目里面建立多个 model ，每个 model 都是一个子 maven 项目

添加 model 后项目根目录的 pom.xml 中发现多了下面几行

    <modules>
        <module>xxx</module>
    </modules>

而且我们新建 model 中也有一个 pom.xml 里面有如下内容

    <parent>
        <artifactId>xxxx</artifactId>
        <groupId>xxxx</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>

这时，可以把一些各个 model 公共的部分移动到父目 pom.xml

## 父 pom.xml 中依赖相关注意
1. 父 pom.xml 中应写入所有 model 的全部依赖.

2. 父 pom.xml 中的依赖应该写入 dependencyManagement 中,如果直接写入 dependencies 中则整个项目的所有 model 都将继承这些依赖，这显然是不恰当的。

        <dependencyManagement>
            <dependencies>
                // 项目的所有依赖，携带版本信息
            </dependencies>
        </dependencyManagement>

3. 子 pom.xml 中依赖那些包还是需要些清楚，但是不要再写 version,scope,exclusions 信息了，version,scope,exclusions 交个全局的父 pom.xml 统一管理,这样就不会出现版本冲突的问题了.

## 版本管理
项目的版本命名为 : x.y.z-里程碑

    x 是架构上的变化，大变化
    y 分支，在大版本上的分支
    z 在这个分支的代码修改，更新

里程碑分为

    SNAPSHOT
    ALPHA
    BETA
    Release(RC) 发行版本
    GA 可靠版本

## maven 生命周期
clean

    pre-clean
    clean
    post-clean
    
compile 实际上可以分为下面四个大步骤(网上粘的)

    prepare-resources 资源拷贝,本阶段可以自定义需要拷贝的资源
    compile	编译,本阶段完成源代码编译 
    package	打包,本阶段根据 pom.xml 中描述的打包配置创建 JAR / WAR 包
    install	安装,本阶段在本地 / 远程仓库中安装工程包
    
更具体的可以分为下面这些步骤, 记住几个关键步骤的顺序就好了。

    validate	检查工程配置是否正确，完成构建过程的所有必要信息是否能够获取到。
    initialize	初始化构建状态，例如设置属性。
    generate-sources	生成编译阶段需要包含的任何源码文件。
    process-sources	处理源代码，例如，过滤任何值（filter any value）。
    generate-resources	生成工程包中需要包含的资源文件。
    process-resources	拷贝和处理资源文件到目的目录中，为打包阶段做准备。
    compile	编译工程源码。
    process-classes	处理编译生成的文件，例如 Java Class 字节码的加强和优化。
    generate-test-sources	生成编译阶段需要包含的任何测试源代码。
    process-test-sources	处理测试源代码，例如，过滤任何值（filter any values)。
    test-compile	编译测试源代码到测试目的目录。
    process-test-classes	处理测试代码文件编译后生成的文件。
    test	使用适当的单元测试框架（例如JUnit）运行测试。
    prepare-package	在真正打包之前，为准备打包执行任何必要的操作。
    package	获取编译后的代码，并按照可发布的格式进行打包，例如 JAR、WAR 或者 EAR 文件。
    pre-integration-test	在集成测试执行之前，执行所需的操作。例如，设置所需的环境变量。
    integration-test	处理和部署必须的工程包到集成测试能够运行的环境中。
    post-integration-test	在集成测试被执行后执行必要的操作。例如，清理环境。
    verify	运行检查操作来验证工程包是有效的，并满足质量要求。
    install	安装工程包到本地仓库中，该仓库可以作为本地其他工程的依赖。
    deploy	拷贝最终的工程包到远程仓库中，以共享给其他开发人员和工程。

1. mvn compile 相当于执行到 complie 的 compile 这一步
2. mvn test 相当于执行到 complie 的 test 这一步
3. mvn package 相当于执行到 complie 的 package 这一步
4. mvn install 相当于执行到 complie 的 install 这一步
5. mvn deploy 相当于执行到 complie 的 deploy 这一步

## 测试
maven 所有的测试都放在 src/test/java 中

默认执行一下三种结构的类的测试

    Test**
    **Test
    **TestCase

测试相关配置，一般不需要修改

参考：[http://maven.apache.org/surefire/maven-surefire-plugin/test-mojo.html](http://maven.apache.org/surefire/maven-surefire-plugin/test-mojo.html)

    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>2.19.1</version>
        <dependencies>
            <dependency>
                <groupId>org.apache.maven.surefire</groupId>
                <artifactId>surefire-junit47</artifactId>
                <version>2.19.1</version>
            </dependency>
        </dependencies>
        <configuration>
          <excludes>
            <exclude>**/TestXXX.java</exclude>
            <exclude>**/TestYYY.java</exclude>
          </excludes>
          <includes>
            <include>**/**HAHA.java</include>
          </includes>
        </configuration>
    </plugin>
    
## 插件使用
参考: [http://maven.apache.org/plugins/](http://maven.apache.org/plugins/)

在父 pom.xml 内部，build 节点写配置 pluginManagement 作用和 dependencyManagement 一样，管理插件的具体配置，configuration 节点的一些参数可以在插件网站上查到，execution 绑定插件在生命周期的哪个阶段执行，以及执行那条命令。

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-source-plugin</artifactId>
                    <version>3.0.1</version>
                    <configuration>
                        <!--http://maven.apache.org/plugins/maven-source-plugin/jar-mojo.html-->
                    </configuration>
                    <executions>
                        <execution>
                            <!--Binds by default to the lifecycle phase: package.-->
                            <phase>package</phase>
                            <goals>
                                <goal>jar</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.6.1</version>
                    <configuration>
                        <source>1.8</source>
                        <target>1.8</target>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>

子 pom.xml 只需要声明 groupId artifactId 就可以使用插件

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

## 常用插件
1. resources
    
    参考: [http://www.cnblogs.com/pixy/p/4798089.html](http://www.cnblogs.com/pixy/p/4798089.html)

    构建Maven项目的时候，如果没有进行特殊的配置，Maven会按照标准的目录结构查找和处理各种类型文件。

        src/main/java和src/test/java 
        这两个目录中的所有*.java文件会分别在comile和test-comiple阶段被编译，编译结果分别放到了target/classes和targe/test-classes目录中，但是这两个目录中的其他文件都会被忽略掉。
         
        src/main/resouces和src/test/resources
        这两个目录中的文件也会分别被复制到target/classes和target/test-classes目录中。
        
        target/classes
        打包插件默认会把这个目录中的所有内容打入到jar包或者war包中。

    但是有时候我们会把有些配置文件常与.java文件一起放在src/main/java目录，这个时候就需要进行一些特殊的配置了，这里就需要用到 resources 插件
    
    配置如下

        <build>
            ...
            <resources>
              <resource>
                <directory>src/my-resources</directory>
                <includes>
                  <include>**/*.txt</include>
                  <include>**/*.rtf</include>
                </includes>
              </resource>
              ...
            </resources>
            ...
        </build>
    
    或者

        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-resources-plugin</artifactId>
            <version>3.0.2</version>
            <configuration>
                <encoding>UTF-8</encoding>
                <outputDirectory>${basedir}/target/classes</outputDirectory>
                <resources>
                    <resource>
                        <directory>${basedir}/src/main/java</directory>
                        <includes>
                            <include>**/*.xml</include>
                        </includes>
                    </resource>
                </resources>
            </configuration>
            <executions>
                <execution>
                    <phase>process-resources</phase>
                    <goals>
                        <goal>resources</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    
2. compiler
    
    参考: [http://maven.apache.org/plugins/maven-compiler-plugin/compile-mojo.html](http://maven.apache.org/plugins/maven-compiler-plugin/compile-mojo.html)

    可以设置项目使用的 jdk版本，compilerArgs 等信息

        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.6.1</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
                <compilerArgs>
                    <arg>-verbose</arg>
                    <arg>-Xlint:all,-options,-path</arg>
                </compilerArgs>
            </configuration>
        </plugin>

3. surefire 测试插件
    
    上面有介绍，参考: [http://maven.apache.org/surefire/maven-surefire-plugin/test-mojo.html](http://maven.apache.org/surefire/maven-surefire-plugin/test-mojo.html)

4. jar

    可以指定打包时排除哪些文件，指定 manifest 文件的内容，或者可以使用

        <manifestFile>${project.build.outputDirectory}/META-INF/MANIFEST.MF</manifestFile>

    直接指定我们编写好的 manifest 文件
    
    archive 配置参看: [http://maven.apache.org/shared/maven-archiver/index.html](http://maven.apache.org/shared/maven-archiver/index.html)

        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <version>3.0.2</version>
            <configuration>
                <excludes>
        　　　　    <exclude>**.properties</exclude>
        　　　　</excludes>
                <archive>
                    <manifest>
                        <addClasspath>true</addClasspath>
                        <classpathPrefix>lib/</classpathPrefix>
                        <mainClass>com.demo.HelloWorld</mainClass>
                    </manifest>
                </archive>
            </configuration>
        </plugin>

5. dependency 配合 jar 一起使用，拷贝依赖到指定的 lib 目录.

    参看: [http://maven.apache.org/plugins/maven-dependency-plugin/copy-dependencies-mojo.html](http://maven.apache.org/plugins/maven-dependency-plugin/copy-dependencies-mojo.html)

        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-dependency-plugin</artifactId>
            <version>3.0.0</version>
            <configuration>
                <outputDirectory>${project.build.directory}/lib</outputDirectory>
            </configuration>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>copy-dependencies</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>

6. assembly
    这篇文章介绍的不错: [http://ee-dreamer.com/?p=1129](http://ee-dreamer.com/?p=1129)

    参考: [http://maven.apache.org/plugins/maven-assembly-plugin/descriptor-refs.html](http://maven.apache.org/plugins/maven-assembly-plugin/descriptor-refs.html)
    
    具体配置

        <plugin>
            <artifactId>maven-assembly-plugin</artifactId>
            <!--<version>3.0.0</version>-->
            <configuration>
                <descriptors>
                    <!--assembly 文件说明大什么类型的包，怎么打包-->
                    <descriptor>src/main/resources/assembly.xml</descriptor>
                </descriptors>
                <archive>
                    <!--应该除了打 jar 包，不会使用吧-->
                    <manifest>
                        <addClasspath>true</addClasspath>
                        <mainClass>com.acyouzi.test.Test</mainClass>
                        <classpathPrefix>/</classpathPrefix>
                    </manifest>
                </archive>
            </configuration>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <!-- single 相关配置 http://maven.apache.org/plugins/maven-assembly-plugin/single-mojo.html-->
                        <goal>single</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    
    assembly 文件在 src/main/resources/assembly.xml
    
    相关参数: [ http://maven.apache.org/plugins/maven-assembly-plugin/assembly.html]( http://maven.apache.org/plugins/maven-assembly-plugin/assembly.html)

        <assembly xmlns="http://maven.apache.org/ASSEMBLY/2.0.0"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.0.0 http://maven.apache.org/xsd/assembly-2.0.0.xsd">
            <formats>
                <!--压缩成什么格式的-->
                <format>tar.gz</format>
            </formats>
            <!--是否在包外面包装一个文件夹-->
            <includeBaseDirectory>true</includeBaseDirectory>
            <fileSets>
                <fileSet>
                    <!--把某个目录下的文件拷贝到包下面的某个目录-->
                    <directory>${project.build.outputDirectory}</directory>
                    <outputDirectory>/lib</outputDirectory>
                    <!--过滤掉一些不需要拷贝的文件-->
                    <excludes>
                        <exclude>*.sh</exclude>
                    </excludes>
                </fileSet>
            </fileSets>
            <dependencySets>
                <dependencySet>
                    <!--把依赖包一块复制到包下的某个文件夹-->
                    <outputDirectory>/lib</outputDirectory>
                    <!--false不解压，是以jar包的形式出现，如果为 true 包会被解压成 class 文件-->
                    <unpack>false</unpack>
                    <!--哪些依赖内容范围的打包，runtime 一般来讲 com.mysql.4j 这类包 scope 是 runtime 如果使用 complie 作用域，这种包就不会复制过去导致错误的发生-->
                    <scope>runtime</scope>
                    <!--排除的依赖-->
                    <excludes>
                        <exclude>org.apache.ant:*:jar</exclude>
                    </excludes>
                </dependencySet>
            </dependencySets>
        </assembly>
        
7. source 用于生成源码包
    
    参考: [http://maven.apache.org/plugins/maven-source-plugin/jar-mojo.html](http://maven.apache.org/plugins/maven-source-plugin/jar-mojo.html)

        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-source-plugin</artifactId>
            <version>3.0.1</version>
            <configuration>
                <outputDirectory>${project.build.directory}</outputDirectory>
                <attach>false</attach>
            </configuration>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>jar-no-fork</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    
8. javadoc
    
    参考: [http://maven.apache.org/plugins/maven-javadoc-plugin/aggregate-mojo.html](http://maven.apache.org/plugins/maven-javadoc-plugin/aggregate-mojo.html)

        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-javadoc-plugin</artifactId>
            <version>2.9.1</version>
            <configuration>
                <!--在一个项目有多个 model 的时候使用-->
                <aggregate>true</aggregate>
                <reportOutputDirectory>${project.build.directory}</reportOutputDirectory>
            </configuration>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>aggregate</goal>
                    </goals>
                </execution>
            </executions>
        </plugin> 

9. Checkstyle

    参考: [http://maven.apache.org/plugins/maven-checkstyle-plugin/checkstyle-aggregate-mojo.html](http://maven.apache.org/plugins/maven-checkstyle-plugin/checkstyle-aggregate-mojo.html)
    
    可以使用 google java style [ http://google.github.io/styleguide/javaguide.html]( http://google.github.io/styleguide/javaguide.html) 做检查
    
    xxxchecks.xml 最好是根据 checkstyle 依赖的版本下载相应的 jar 包，然后从 jar 包中提取 xxxchecks.xml 再根据需要做一定修改 , 因为不同版本的关键字相差很多,版本不匹配容易发生令人头疼的问题。

        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-checkstyle-plugin</artifactId>
            <version>2.17</version>
            <configuration>
                <includeTestResources>false</includeTestResources>
                <!--<includes>**/*.java</includes>-->
                <resourceIncludes>**/*.properties</resourceIncludes>
                <configLocation>src/main/resources/google_checks.xml</configLocation>
                <outputFile>${project.build.directory}/checkstyle-result.plain</outputFile>
                <!--"plain" and "xml"-->
                <outputFileFormat>plain</outputFileFormat>
                <consoleOutput>false</consoleOutput>
            </configuration>
            <dependencies>
                <dependency>
                    <groupId>com.puppycrawl.tools</groupId>
                    <artifactId>checkstyle</artifactId>
                    <version>7.1.2</version>
                </dependency>
            </dependencies>
            <executions>
                <execution>
                    <phase>validate</phase>
                    <goals>
                        <goal>checkstyle</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>

10. cobertura 是一款统计测试覆盖率的工具 官网地址是 [http://www.mojohaus.org/cobertura-maven-plugin/](http://www.mojohaus.org/cobertura-maven-plugin/)

## nexus 相关
mvn:deploy 提交命令，提交都是给 nexus hosted 类型的工厂

hosted 类型的工厂有三个，hosted 工厂都是对内服务的，不会面向公网
    
    3rd party nexus 库中没有，从官网下载的 jar ,提交到这个库来管理
    Releases 我们提交的  release 版本的 jar
    Snapshots 我们提交的  snapshots 版本的 jar

proxy 类型的工厂，比如：
    
    Central 工厂，从中央工厂下载下来的数据。

group 类型工厂，会自动从 3rd party ，Releases，Snapshots，Central 这些工厂查找数据，一般我们使用这种类型的工厂地址作为仓库
    


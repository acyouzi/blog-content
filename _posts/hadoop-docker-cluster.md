---
title: 使用 docker 搭建 hadoop ha 集群
date: 2016-11-01 22:54:27
author: "acyouzi"
cdn: header-off
header-img: img/docker.png
tags:
	- docker
	- hadoop
    - 大数据
---
### 准备材料
首先为什么要使用Docker来搭建集群呢，因为我是在本机测试搭建hadoop集群，在本机性能不是超强的情况下，开七八个虚拟机的确还是比较困难的事情。所在这里使用Docker来搭建，相比于开多台虚拟机，使用Docker对机器的要求显然要低的多

首先我们需要准备的材料有如下几项：
* jdk-8u102-linux-x64.tar.gz
* zookeeper-3.4.9.tar.gz
* hadoop-2.7.3.tar.gz

以上是我在配置集群时的最新版本，当然也可以自行选择其他版本。

然后新建一个目录 ~/docker-file 把上面的安装包都拷贝进去

    cd ~
    mkdir docker-file

其次需要准备ssh的秘钥，用于配置秘钥登录

    ssh-keygen -t rsa -b 2048
    cp ~/.ssh/* ~/docker-file/

准备 config 文件，config 文件用于配置ssh不用在第一次登录时输入yes确认，config 文件内容如下

    StrictHostKeyChecking no
    # 每次都是新的 know host 文件
    UserKnownHostsFile /dev/null

### 安装Docker
1. 首先看看内核版本，貌似不要低于3.10就好，一般ubuntu 14.04是没啥问题的
    
        uname -a

2. 添加apt源

        apt-get install apt-transport-https ca-certificates
        apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80	--recv-keys	58118E89F3A912897C070ADBF76221572C52609D
        apt-get update
        apt-get install -y linux-image-extra-$(uname -r)
        apt-get install -y docker-engine

3. 安装Docker

        apt-get	install	-y	docker-engine
        # 测试一下看是否安装成功
        docker -v

4. 安装pipework

        git clone https://github.com/jpetazzo/pipework
        cp ~/pipework/pipework /usr/local/bin/

5. 安装 bridge-utils，配置网桥

        brctl addbr br0
        brctl stp br1 off
        ifconfig br0 192.168.0.1/24
        
### 编写Dockerfile
1. pull docker image
    
        # 可以去 hub.docker.com 上选择合适的镜像，但是可能网速比较慢
        # 这里我们从网易蜂巢的仓库中选一个ubuntu的镜像,镜像中默认集成了openssh-server
        # 下载镜像
        docker pull hub.c.163.com/public/ubuntu:16.04
        # 查看镜像 
        docker images

2. 编写Dockerfile

        # 指定基于哪个镜像创建docker
        FROM hub.c.163.com/ubuntu:16.04
        MAINTAINER acyouzi <acyouzi@acyouzi.com>
        
        # 切换工作目录
        WORKDIR /opt/

        # jdk 安装
        COPY jdk-8u102-linux-x64.tar.gz /opt/
        RUN tar -xf jdk-8u102-linux-x64.tar.gz
        RUN echo "export JAVA_HOME='/opt/jdk1.8.0_102'" >> /etc/profile
        RUN echo "export PATH=\$PATH:\$JAVA_HOME/bin" >> /etc/profile 
        
        # zookeeper-3.4.9
        COPY zookeeper-3.4.9.tar.gz /opt/
        RUN tar -xf zookeeper-3.4.9.tar.gz
        RUN echo "export ZOOKEEPER_HOME='/opt/zookeeper-3.4.9'" >> /etc/profile
        RUN echo "export PATH=\$PATH:\$ZOOKEEPER_HOME/bin" >> /etc/profile

        # hadoop-2.7.3
        COPY hadoop-2.7.3.tar.gz /opt/
        RUN tar -xf hadoop-2.7.3.tar.gz
        RUN echo "export HADOOP_HOME='/opt/hadoop-2.7.3'" >> /etc/profile
        RUN echo "export PATH=\$PATH:\$HADOOP_HOME/bin:\$HADOOP_HOME/sbin" >> /etc/profile
        
        # add ssh key
        RUN mkdir /root/.ssh
        COPY id_rsa /root/.ssh/
        COPY id_rsa.pub /root/.ssh/
        COPY config /root/.ssh/
        WORKDIR /root/.ssh
        RUN cat id_rsa.pub >> authorized_keys

        # sshd -D
        ENTRYPOINT /usr/sbin/sshd -D

        WORKDIR /

### build dockerfile

    docker build ~/docker-file -t bigdata:1.0.0

    # 查看镜像，会发现多了一个bigdata:1.0.0
    docker images

    # 测试创建容器
    docker run -itd --name=test bigdata:1.0.0

    # 查看运行的容器
    docker ps

    # 查看docker ip 地址
    docker inspect -f '{{ .NetworkSettings.IPAddress }}' test

    # ssh 远程登录
    ssh root@<ip>
    
    # 登录之后可以尝试运行几个命令看看软件是否安装成功
    java -version
    hadoop version

### 集群规划

首先需要确定有哪几种类型的节点，以及节点数量
* zookeeper 集群，由3个节点组成，同时会成为journalnode
* resourcemanager 两个节点
* namenode 两个节点
* datanode 三个节点

这样我们节点就由:
> zookeeper1,zookeeper2,zookeeper3
> resourcemanager1,resourcemanager2
> namenode1,namenode2
> datanode1,datanode2,datanode3
这十个节点组成。

### 准备配置文件
根据我们的节点规划来编写对应的配置文件

0. 新建文件夹 cluster-build，下面准备的所有文件都放到该目录下

        mkdir ~/cluster-build 

1. 从 hadoop安装目录下的 /etc/hadoop/ 目录下拷贝 hadoop_env.sh 到 ~/cluster-build/

        # 修改 hadoop_env.sh 文件的 JAVA_HOME 修改为dockerfile里面java安装的目录
        export JAVA_HOME='/opt/jdk1.8.0_102'

2. core-site.xml

        <configuration>
        <!-- set nameservice -->
        <property>
            <name>fs.defaultFS</name>
            <value>hdfs://acyouzi/</value>
        </property>
        <!-- hadoop tmp dir -->
        <property>
            <name>hadoop.tmp.dir</name>
            <value>/root/hadoop</value>
        </property>
        <!-- set zookeeper addr -->
        <property>
            <name>ha.zookeeper.quorum</name>
            <value>zookeeper1:2181,zookeeper2:2181,zookeeper3:2181</value>
        </property>
        </configuration>

3. mapred-site.xml

        <configuration>
        <property>
            <name>mapreduce.framework.name</name>
            <value>yarn</value>
        </property>
        </configuration> 

4. hdfs-site.xml

        <configuration>
        <!--nameservice 配置为acyouzi,与core-site.xml中统一 -->
        <property>
            <name>dfs.nameservices</name>
            <value>acyouzi</value>
        </property>
        <!-- nameservice下边的namenode地址，两个 -->
        <property>
            <name>dfs.ha.namenodes.acyouzi</name>
            <value>namenode1,namenode2</value>
        </property>
        <!-- RPC地址 -->
        <property>
            <name>dfs.namenode.rpc-address.acyouzi.namenode1</name>
            <value>namenode1:9000</value>
        </property>
        <!-- http地址 -->
        <property>
            <name>dfs.namenode.http-address.acyouzi.namenode1</name>
            <value>namenode1:50070</value>
        </property>
        <property>
            <name>dfs.namenode.rpc-address.acyouzi.namenode2</name>
            <value>namenode2:9000</value>
        </property>
        <property>
            <name>dfs.namenode.http-address.acyouzi.namenode2</name>
            <value>namenode2:50070</value>
        </property>
        <!-- 放在JournalNode的哪里-->
        <property>
            <name>dfs.namenode.shared.edits.dir</name>
            <value>qjournal://zookeeper1:8485;zookeeper2:8485;zookeeper3:8485/acyouzi</value>
        </property>
        <!-- JournalNode本地数据放置位置 -->
        <property>
            <name>dfs.journalnode.edits.dir</name>
            <value>/root/journaldata</value>
        </property>
        <!-- namenode自动切换 -->
        <property>
            <name>dfs.ha.automatic-failover.enabled</name>
            <value>true</value>
        </property>
        <!-- 失败自动切换方式，官方文档中有详细介绍 -->
        <property>
            <name>dfs.client.failover.proxy.provider.acyouzi</name>
            <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
        </property>
        <!-- fencing 隔离配置 -->
        <property>
            <name>dfs.ha.fencing.methods</name>
            <value>
            sshfence
            shell(/bin/true)
            </value>
        </property>
        <!-- fence隔离 ssh 配置 -->
        <property>
            <name>dfs.ha.fencing.ssh.private-key-files</name>
            <value>/root/.ssh/id_rsa</value>
        </property>
        <property>
            <name>dfs.ha.fencing.ssh.connect-timeout</name>
            <value>30000</value>
        </property>
        </configuration>

5. yarn-site.xml

        <configuration>
        <property>
            <name>yarn.resourcemanager.ha.enabled</name>
            <value>true</value>
        </property>
        <!-- 指定RM的cluster id -->
        <property>
            <name>yarn.resourcemanager.cluster-id</name>
            <value>yrc</value>
        </property>
        <!-- resourcemanager 名字和地址配置 -->
        <property>
            <name>yarn.resourcemanager.ha.rm-ids</name>
            <value>rm1,rm2</value>
        </property>
        <property>
            <name>yarn.resourcemanager.hostname.rm1</name>
            <value>resourcemanager1</value>
        </property>
        <property>
            <name>yarn.resourcemanager.hostname.rm2</name>
            <value>resourcemanager2</value>
        </property>
        <!-- zk cluster -->
        <property>
            <name>yarn.resourcemanager.zk-address</name>
            <value>zookeeper1:2181,zookeeper2:2181,zookeeper3:2181</value>
        </property>
        <property>
            <name>yarn.nodemanager.aux-services</name>
            <value>mapreduce_shuffle</value>
        </property>
        </configuration>

6. zoo.cfg
        
        tickTime=2000
        initLimit=10
        syncLimit=5
        
        # 数据存放目录 myid会放在这下面
        dataDir=/root/zookeeper
        clientPort=2181
        
        # zk 集群地址
        server.1=zookeeper1:2888:3888
        server.2=zookeeper2:2888:3888
        server.3=zookeeper3:2888:3888


7. 编写 zk-id.sh 脚本

        #!/bin/bash
        echo $1 > /root/zookeeper/myid

    修改权限为可以执行

        chmod 775 zk-id.sh

### 编写hadoop集群脚本
1. 在 ~/cluster-build 目录下创建 hadoop-cluster.sh 文件

        #!/bin/bash
        
        # 定义用于拷贝配置文件，分配ip地址的函数
        function cp_config(){
            docker exec ${1}${2} cp /root/cluster-build/hadoop-env.sh /opt/hadoop-2.7.3/etc/hadoop/
            docker exec ${1}${2} cp /root/cluster-build/core-site.xml /opt/hadoop-2.7.3/etc/hadoop/
            docker exec ${1}${2} cp /root/cluster-build/hdfs-site.xml /opt/hadoop-2.7.3/etc/hadoop/
            docker exec ${1}${2} cp /root/cluster-build/yarn-site.xml /opt/hadoop-2.7.3/etc/hadoop/
            docker exec ${1}${2} cp /root/cluster-build/mapred-site.xml /opt/hadoop-2.7.3/etc/hadoop/
            
            docker exec ${1}${2} cp /root/cluster-build/hosts /etc
            pipework br0 ${1}${2} 192.168.0.$((${3}+${2}-1))/24
        }

        # 删除上次创建的hosts配置
        test -e hosts&& rm hosts
        
        # 定义集群数量和集群起始ip分配地址
        declare -i zk_num=3
        declare -i zk_ipbase=2
        declare -i rm_num=2
        declare -i rm_ipbase=20
        declare -i nn_num=2
        declare -i nn_ipbase=40
        declare -i dn_num=3
        declare -i dn_ipbase=60
        
        # hosts configration
        echo 127.0.0.1 localhost.localdomain localhost >> hosts
        for ((i=1; i<=${zk_num}; i=i+1 ))
        do
            echo 192.168.0.$((${zk_ipbase}+${i}-1))    zookeeper${i} >> hosts
        done
        for ((i=1; i<=${rm_num}; i=i+1 ))
        do
            echo 192.168.0.$((${rm_ipbase}+${i}-1))    resourcemanager${i} >> hosts
        done
        for ((i=1; i<=${nn_num}; i=i+1 ))
        do
            echo 192.168.0.$((${nn_ipbase}+${i}-1))    namenode${i} >> hosts
        done
        for ((i=1; i<=${dn_num}; i=i+1 ))
        do
            echo 192.168.0.$((${dn_ipbase}+${i}-1))    datanode${i} >> hosts
        done

        # zookeeper
        for ((i=1; i<=${zk_num}; i=i+1 ))
        do
            docker run -itd -h zookeeper${i} --name=zookeeper${i} -v /root/cluster-build:/root/cluster-build bigdata:1.0.2
            docker exec zookeeper${i} mkdir /root/zookeeper
            docker exec zookeeper${i} /root/cluster-build/myid.sh ${i}
            docker exec zookeeper${i} cp /root/cluster-build/zoo.cfg /opt/zookeeper-3.4.9/conf
            cp_config 'zookeeper' ${i} ${zk_ipbase}
        done
        
        # resourcemanager
        # slaves 文件在启动集群时会用到
        test -e slaves && rm slaves
        for ((i=1; i<=${rm_num}; i=i+1 ))
        do
            echo resourcemanager${i} >> slaves
        done
        for ((i=1; i<=${rm_num}; i=i+1 ))
        do
            docker run -itd -h resourcemanager${i} --name=resourcemanager${i} -v /root/cluster-build:/root/cluster-build bigdata:1.0.2
            docker exec resourcemanager${i} cp /root/cluster-build/slaves /opt/hadoop-2.7.3/etc/hadoop/
            cp_config 'resourcemanager' ${i} ${rm_ipbase}
        done

        # namenode
        for ((i=1; i<=${nn_num}; i=i+1 ))
        do
            docker run -itd -h namenode${i} --name=namenode${i} -v /root/cluster-build:/root/cluster-build bigdata:1.0.2
            cp_config 'namenode' ${i} ${nn_ipbase}
        done

        # datanode
        test -e slaves && rm slaves
        for ((i=1; i<=2; i=i+1 ))
        do
            echo datanode${i} >> slaves
        done
        for ((i=1; i<=3; i=i+1 ))
        do
            docker run -itd -h datanode${i} --name=datanode${i} -v /root/cluster-build:/root/cluster-build bigdata:1.0.2
            docker exec datanode${i} cp /root/cluster-build/slaves /opt/hadoop-2.7.3/etc/hadoop/
            cp_config 'datanode' ${i} ${dn_ipbase}
        done

2. 编写删除集群脚本
    
        docker stop $(docker ps -q)
        docker rm $(docker ps -a -q)

### 启动集群
可以把生产hosts文件中的内容复制到本机的 /etc/hosts 文件中
    
    cat hosts >> /etc/hosts

1. 启动zookeeper集群

        ssh root@zookeeper1
        zkServer.sh start

        ssh root@zookeeper2
        zkServer.sh start
        
        ssh root@zookeeper3
        zkServer.sh start
        
        # 运行 jps 命令 看一下是不是多了一个java线程
        jps
        # 查看状态
        zkServer.sh status

2. 启动 journalnode 集群，如果失败可以去 /opt/hadoop-2.7.3/logs/ 目录下查看输出
        
        ssh root@zookeeper1
        hadoop-daemon.sh start journalnode

        ssh root@zookeeper1
        hadoop-daemon.sh start journalnode

        ssh root@zookeeper1
        hadoop-daemon.sh start journalnode

3. namenode 格式化

        ssh root@namenode1
        hdfs namenode -format

    为了保持一致，直接把 namenode1 的 /root/hadoop/ 目录下的内容拷贝到 namenode2中

        scp -r /root/hadoop/ namenode2:/root/

    在其中一个namenode节点上运行下面的命令，会在 zookeeper 集群中创建一个 /acyouzi 目录

        hdfs zkfc -formatZK
    
4. 启动datanode

        ssh root@datanode1
        # 会根据 /etc/hadoop/slaves 文件启动所有的datanode
        start-dfs.sh

5. 启动resourcemanager
        
        ssh root@resourcemanager1
        # 会根据 /etc/hadoop/slaves 文件启动所有的启动resourcemanager
        start-yarn.sh

6. 打开浏览器输入 namenode1:50070 namenode2:50070 查看节点状态
7. 往hdfs中放入文件试试

        hadoop fs -put ~/test.md /test

### 注意事项
1. 在创建 container 的时候务必通过 -h 参数指定主机名，否则启动journalnode节点时会报错，大概是 unknowhost 错误
2. 我shell写的真的很烂
3. 还缺一个启动脚本，在通过docker start $(docker ps -a -q) 启动时给容器分配ip,还没想好怎么写
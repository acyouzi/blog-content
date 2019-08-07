---
title: Docker 容器与容器云-笔记-01-Docker 基本使用
date: 2017-01-12 23:26:14
author: "acyouzi"
description: 
tags:
    -  容器
---
以下内容是 Docker 容器与云的前半部分。后半部分正在研读中，kubernetes 的设计思想真的很让人佩服。每个人，哪怕不用这套东西，也应该学习一下他的设计思想.

### 首先是修补一个问题
问题：当我们在 ubuntu 从官方源安装完 docker 后，如果尝试修改 /etc/default/docker 配置文件来改变 Docker daemon 启动参数时发现并不起作用。

原因是 /lib/systemd/system/docker.service 文件压根就没有引用EnvironmentFile
修改 docker.service 的 [Service] 部分的如下两行

	EnvironmentFile=-/etc/default/docker
	ExecStart=/usr/bin/dockerd $DOCKER_OPTS -H fd://

然后重启容器

	systemctl daemon-reload
	systemctl restart docker

### Docker 基本知识
1. docker 使用 namespace 来实现资源隔离，使用 cgroup 做资源限制，namespace 有如下几种
		
		root@ubuntu:~# ll /proc/$$/ns/
		total 0
		dr-x--x--x 2 root root 0 Jan 16 00:33 ./
		dr-xr-xr-x 9 root root 0 Jan 16 00:21 ../
		lrwxrwxrwx 1 root root 0 Jan 16 00:33 ipc -> ipc:[4026531839]
		lrwxrwxrwx 1 root root 0 Jan 16 00:33 mnt -> mnt:[4026531840]
		lrwxrwxrwx 1 root root 0 Jan 16 00:33 net -> net:[4026531957]
		lrwxrwxrwx 1 root root 0 Jan 16 00:33 pid -> pid:[4026531836]
		lrwxrwxrwx 1 root root 0 Jan 16 00:33 user -> user:[4026531837]
		lrwxrwxrwx 1 root root 0 Jan 16 00:33 uts -> uts:[4026531838]

2. 容器的文件系统采用联合挂载，有分层挂载写时复制，内容寻址，联合挂载等几种特性。

3. docker 镜像实际上可以分为三层,默认存放在 /var/lib/docker/image 目录下
	
	可读写部分
	init-layer
	只读层

### Docker 存储驱动
docker 支持多种存储驱动，有 devicemapper、aufs、vfs、overlay 等，默认使用 aufs，支持联合挂载( 将不同目录挂载到同一个目录上，这个过程对用户是透明的。一般可读写层在上，下层是只读层)，overlay 是一种新型联合文件系统，理论上性能好于 aufs

/var/lib/docker/image/aufs 用于存放镜像相关元数据
/var/lib/docker/aufs 下边有 diff layers mnt 三个目录

	/mnt 为 aufs 挂载目录
	/diff 为实际数据来源，包括只读层和可读可写层
	/layers 为每层依赖的层描述文件 就是这个层依赖哪些层

在容器启动时会在 /mnt 和 /diff 目录下见到 /< mountID >-init 层，这个就是上面提到的镜像三层中的 init-layer

	/< mountID >-init 层作为最后一层只读层，用于挂载并重新生成以下文件
		/dev/pts
		/dev/shm
		/proc
		/sys
		/.dockerinit
		/.dockerenv
		/etc/resolve.conf
		/etc/host
		/etc/hostname
		/dev/console
		/etc/mtab

因为这些文件与容器内的环境息息相关，不适合作为打包镜像文件内容，所以这层docker 做了特殊处理，只在启动时自动添加，并且利用用户配置自动生成内容，只用容器在运行过程中被改动，并且 commit 了，才会持久化，否则这一层内容不会被保存。

#### 存储驱动实验
1. 找到联合挂载中的可读可写层

		查看短ID
		docker ps
		cd /var/lib/docker/ 
		
		查看 mountID
		cat image/aufs/layerdb/mounts/2313d627f4df7b7cec52cd8eb1cd9c628fedfa8b93cbf5095de87d3d9958fc7a/mount-id
		
		得到 mountID 为 4a6f315dfb32c50e7ff54589d52f0f93acf1a74fa50c381bb35d54c6bbb41e3e
		然后找到镜像挂载位置
		cd /var/lib/docker/aufs/mnt/4a6f315dfb32c50e7ff54589d52f0f93acf1a74fa50c381bb35d54c6bbb41e3e 
		
		查看大小， 在容器启动的状态下挂载文件
		du --max-depth=1 -h ./aufs/mnt | grep 4a6f315dfb3 

		进入容器，在root 目录下创建一个文件，然后关闭容器
		docker exec -it test /bin/bash
		
		停止后查看镜像挂载目录
		发现 /var/lib/docker/aufs/mnt/4a6f315dfb32c50e7ff54589d52f0f93acf1a74fa50c381bb35d54c6bbb41e3e/  目录空了
		/var/lib/docker/aufs/diff/4a6f315dfb32c50e7ff54589d52f0f93acf1a74fa50c381bb35d54c6bbb41e3e/ 下有一个 root/ 目录，目录下边是我们修改的文件

	这就是联合挂载的效果

2. init-layer 效果查看
	
		启动容器，进入容器，修改 /etc/hosts 文件。
		
		然后进入 /var/lib/docker/aufs/diff/4a6f315dfb32c50e7ff54589d52f0f93acf1a74fa50c381bb35d54c6bbb41e3e/ 
		
		发现 这里并没有 etc/hosts 文件
		
		然后进入 /var/lib/docker/aufs/diff/4a6f315dfb32c50e7ff54589d52f0f93acf1a74fa50c381bb35d54c6bbb41e3e-init/
		
		发现存在 etc/hosts 文件，但是没有任何内容。
		
		而且重启容器之后，修改被还原了。

	这是 init 层的效果




### 数据卷 volume
docker volume 命令

	create      Create a volume
	inspect     Display detailed information on one or more volumes
	ls          List volumes
	rm          Remove one or more volumes

创建容器的时候使用 -v 挂载数据卷,一个容器可挂载多个 volume

	-v /xxx 自动创建一个 volume 并挂载到容器中
	-v volume_name:/xxx 挂载一个已经存在的 volume 
	-v /host/dir:/xxx 挂载一个本地目录到容器 
	--volume-from container-xx  挂载 container-xx 挂载的全部 volume

dockerfile 中挂载 volume 不能指定要挂载的外部文件。为了可移植性，只能自动创建。

### --link 网络
容器创建时可以通过 --link 指定关联的容器，然后 Docker 会自动维护映射关系，但是只能链接已经存在的容器，所以使用 link 作为容器间通信的方法必须注意启动顺序。

	--link container:alies

但是如果自己创建一个网络把容器网络设置为自己创建的网络，在创建容器 link 时就能使用原本不存在的容器了。这里貌似是因为 Docker 更改了处理 link 的方式。

	docker network create nw-link-test
	docker run -it --name link-test-01 --network nw-link-test  --link link-test-02:link-test-02 ubuntu-apt-163:16.04 /bin/bash
	docker run -it --name link-test-02 --network nw-link-test  --link link-test-01:link-test-01 ubuntu-apt-163:16.04 /bin/bash


### 容器网络
1. libnetwork 支持5种内置驱动
	
		bridge 驱动
		host 驱动 
			这种模式下容器没有网络协议栈，没有独立的 network namespace
		overlay 驱动 
			使用 vxlan 方式，适合 SDN ， 需要额外配置一个存储服务，如 etcd zookeeper
		remote驱动
			调用用户自己实现的网络驱动的插件
		null 驱动
			这种模式下有自己的 network namespace, 但是没有进行任何网络配置

2. 使用 docker network 管理网络
	
		connect     Connect a container to a network
		create      Create a network
			可以指定驱动类型，默认是 bridge 
			创建完，通过 brclt show 可以发现多了一个网桥
		disconnect  Disconnect a container from a network
		inspect     Display detailed information on one or more networks
		ls          List networks
		rm          Remove one or more networks

	
#### network 测试
创建两个网络

	docker network create nw-01
	docker network create nw-02

创建三个容器分别加入两个网络

	docker run -idt --name test00 --network nw-01 ubuntu:16.04
	docker run -idt --name test01 --network nw-01 ubuntu:16.04
	docker run -idt --name test02 --network nw-02 ubuntu:16.04

观察容器 ip test00 test01 在一个网段，能相互ping通，test02 ping 不通 test00 test01

	test00 172.19.0.2
	test01 172.19.0.3
	test02 172.20.0.2

把 test02 加入到 nw-01

	docker network connect nw-01 test02
	然后 test02 多了一块网卡 eth1 ip 172.19.0.4

可以查看 route 规则,在宿主机上通过 route 命令查看路由表

	route -n 

	172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
	172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-91694b4f37a8
	172.19.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-63fb8f706b58
	172.20.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-e63a8cb0655b


#### route 命令使用
直接在命令行下执行route命令来添加路由，不会永久保存，当网卡重启或者机器重启之后，该路由就失效了,可以在/etc/rc.local中添加route命令来保证该路由设置永久有效

	显示当前路由 
	route 
	route -n
	
	Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
	0.0.0.0         192.168.192.2   0.0.0.0         UG    0      0        0 ens33
	172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0

	U Up表示此路由当前为启动状态
	H Host，表示此网关为一主机
	G Gateway，表示此网关为一路由器
	! 表示此路由当前为关闭状态

	route add -net 192.56.76.0 netmask 255.255.255.0 dev eth0
	添加一条route 规则，目标地址是 192.56.76.0 使用设备 eth0

	route del -net 192.56.76.0 netmask 255.255.255.0
	删除一条路由

	route add -net 10.0.0.0 netmask 255.0.0.0 reject
	拒绝转发一条请求

	route add default gw 192.168.192.2
	添加网关 网关是往外部发送的地址，default 代表 0.0.0.0 
	Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
	default         192.168.192.2   0.0.0.0         UG    0      0        0 ens33

	route del default
	删除网关

#### docker daemon 网络启动参数
docker daemon 启动时，可以指定的网络配置, 修改 /etc/default/docker 文件的 DOCKER_OPTS

	--bip=172.17.0.1/24 设置 docker0 的 ip 地址和子网范围
	--fixed-cidr=172.17.0.1/25 设置容器的 ip 获取范围，默认是 docker0 的子网范围
	--mtu=BYTES 

也可以自己创建网桥然后在 daemon 启动的时候，使用 --bridge=xxx 来使用指定网桥

	brclt addbr br0
	ifconfig br0 xxx.xxx.xxx.xxx

也可以跟宿主机的物理网卡相连接
	
	brclt addif br0 ens33

#### iptable 规则
使用 iptables-save 命令打印 iptables rules . iptables-save — dump iptables rules to stdout

容器与外部通信

	-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
	从源地址 172.17.0.0/16 发出，但不是从 docker0 网卡发出的做 snat (源地址转换)

	另外涉及多个网卡之间信息的转发，需要设置 ip_forward 参数为 1
	echo 1 > /proc/sys/net/ipv4/ip_forward

容器之间通信

	-A FORWARD -i docker0 -o docker0 -j ACCEPT

相关参数 daemon 规则 

	--iptables=true 是否允许修改宿主机的 iptable
	--icc=true 是否允许容器间相互通信
	--ip-forward=true 是否开启 ip_forward 功能
	-h hostname   "/etc/hostname"
	--dns=ipAddr DNS 服务器 "/etc/resolv.conf"




### 磁盘限额

	--store-opt 选项只对 devicemapper 文件系统有效
	
* 可以通过创建虚拟文件系统来对磁盘进行限额

		创建文件
		dd if=/dev/zero of=./disk-quota.ext4 count=2048 bs=1MB
		格式化
		mkfs -t ext4 -F  disk-quota.ext4
		挂载
		mount -o loop,rw,usrquota,grpquota ./disk-quota.ext4 ./test/

### 流量限制

	trafic controller 

### 容器化思维
容器就是一个进程，只不过拥有相对独立的环境.把容器当成一个进程来对待，这样很容易理解 kubernetes 这个系统的很多概念了

### 使用 ip 命令操作 network namespace

	ip netns list
	ip netns add NAME
		会在 /var/run/netns/ 目录下建立一个同名文件
	ip netns set NAME NETNSID
	ip [-all] netns delete [NAME]
	ip netns identify [PID]
	ip netns pids NAME

	ip [-all] netns exec [NAME] cmd ...
		在 namespace 里面执行命令
	
	ip netns monitor
	ip netns list-id

	可以进入到 namespace 里面执行命令，使用 exit 退出
	ip netns exec ns-test /bin/bash 

#### ip 命令配置网卡 联通两个 network namespace

	启动设备，在新创建的 network namespace 里面有一个 lo 设备没有启动，最好手动启动
	ip link set dev lo up 

	在宿主机上创建 一对网卡
	ip link add veth-a type veth peer name veth-b
	
	ip link show 
		53: veth-b@veth-a: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
			link/ether 0a:b3:58:bc:b8:75 brd ff:ff:ff:ff:ff:ff
		54: veth-a@veth-b: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
			link/ether 2e:d4:69:1d:e8:95 brd ff:ff:ff:ff:ff:ff
	
	把其中一张网卡加入到 ns-test
	ip link set veth-b netns ns-test

	查看 ns-test 里面的网络设备
	ip netns exec ns-test ip link show 
		1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1
			link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
		53: veth-b@if54: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
			link/ether 0a:b3:58:bc:b8:75 brd ff:ff:ff:ff:ff:ff link-netnsid 0
	
	分别启动 veth-a veth-b 
	ip addr add 10.0.0.1/24 dev veth-a
	ip link set veth-a up

	ip netns exec ns-test ip addr add 10.0.0.1/24 dev veth-b
	ip netns exec ns-test ip link set veth-b up

	验证连通
	ip netns exec ns-test ping 10.0.0.1

#### docker 中的 network namespace
linux 下每个进程都有一个特定的 network namespace 位于 $/proc/$pid/ns 目录下 

	lrwxrwxrwx 1 root root 0 Jan 13 06:25 net -> net:[4026531957]

我们可以通过把docker 进程的这个文件挂载到 /var/run/netns/ 目录下来使用 ip 命令管理这个文件
过程

	新建一个容器 
	docker run -idt --name ns-test --network=none ubuntu:16.04

	查看PID 
	docker inspect --format '{{ .State.Pid }}' ns-test

	建立链接
	ln -s /proc/18638/ns/net /var/run/netns/docker-ns-00

	可以发现 多了一个 namespace
	ip netns list 
	ip netns exec docker-ns-00 ip link show

	在宿主机上创建 一对网卡
	ip link add veth-a type veth peer name veth-b

	brctl addbr br0
	ip addr add 10.0.0.1/24 dev br0
	brctl addif br0 veth-a
	ip link set veth-a up


	把其中一张网卡加入到 ns-test
	ip link set veth-b netns docker-ns-00
	ip netns exec docker-ns-00 ip link set dev veth-b name eth0
	ip netns exec docker-ns-00 ip link set eth0 up
	ip netns exec docker-ns-00 ip addr add 10.0.0.2/24 dev eth0
	ip netns exec docker-ns-00 ip route add default via 10.0.0.1

这就是手动配置 Docker 网络的基本方法，可以看到步骤是非常繁琐的，使用 pipework 可以更简单的完成网络配置。

### pipework 使用
#### 简单使用

	pipework br0 container-name 172.17.2.24/24@172.17.2.1
	@ 后是网关

#### macvlan 链接本地网络
macvlan 是从网卡上虚拟出来的一块新网卡，他和主网卡有不同的 mac 地址，可以独立配置 ip

	pipework ens33 test02 192.168.192.134/24@192.168.192.2

	1. 从主机 ens33 上创建一块 macvlan 网卡 ，然后添加到 test02 中
		ip link add link "$IFNAME" dev "$GUEST_IFNAME" mtu "$MTU" type macvlan mode bridge
	2. 设置 ip 网关  
	
因为 macvlan 设备的进出口流量被 macvlan 隔离，主机不能通过 ens33 访问 macvlan 设备。

解决方法：新建一个 macvlan 把ip 转移到新建的 macvlan 上

		ip link add link ens33 dev eth0m type macvlan mode bridge
		ip link set eth0m up
		ip addr del 192.168.192.132/24 dev ens33
		ip addr add 192.168.192.132/24 dev eth0m
			
#### 支持 dhcp

		pipework eth0 container-name dhcp

#### 支持 Open vSwitch
Open vSwitch 是一个开源的虚拟交换机，创建 Open vSwitch 网桥需要网桥名以 ovs 开头
	
	ovs-vsctl add-br "$IFNAME"
	ovs-vsctl add-port "$IFNAME" "$LOCAL_IFNAME" ${VLAN:+tag="$VLAN"}

#### 支持 vlan
只有 Open vSwitch 和 macvlan 支持划分 vlan，普通 linux 网桥不可

	pipework osv-br0 container dhcp @12
	@ 后面是 vlan 号




### 跨主机通信
#### 桥接
容器占用主机IP,大量Docker容器可能引起广播风暴
每台机器上的 Docker 容器可能获得相同的IP地址，可以通过 --fixed-cidr 参数限定不同主机的所在的网段。

方法是将 eth1 桥接到 docker0 上

	可以通过命令
		brclt addif docker0 eth1
	也可以写在配置文件中
		auto docker0
		Iface docker0 inet static
			address xxx.xxx.xxx.xxx
			netmask xxx
			bridge_ports eth1
			bridge_stp off
			bridge_fd 0
		
#### 直接路由
在主机中添加静态路由来实现，主机中的 docker 需要在不同的子网
配置主机 A 的 docker 在 172.17.1.1/24，主机 B 的 docker 在 172.17.2.1/24 
在主机 A 上添加路由 目的地地址为 172.17.2.0/24 的包转发到 B
在主机 B 上添加路由 目的地地址为 172.17.1.0/24 的包转发到 A

	# 主机 A
	ip addr del 172.17.0.1/16 dev docker0
	ip addr add 172.17.1.1/24 dev docker0
	route add  -net 172.17.2.0/24  gw 192.168.192.133

	iptables -t nat -F POSTROUTING
	iptables -t nat -A POSTROUTING -s 172.17.1.0/24 ! -d 172.17.0.0/16  -j MASQUERADE

	# 主机 B
	ip addr del 172.17.0.1/16 dev docker0
	ip addr add 172.17.2.1/24 dev docker0
	route add -net 172.17.1.0/24 gw 192.168.192.132
	
	iptables -t nat -F POSTROUTING
	iptables -t nat -A POSTROUTING -s 172.17.2.0/24 ! -d 172.17.0.0/16 -j MASQUERADE

### ovs 划分 vlan
如果不划分 vlan 当网络中机器足够多时会导致广播风暴。

#### vlan 相关概念
	
1. access 端口： access 端口都会分配一个 vlan id ，标示他所连接的设备属于哪个 vlan .当数据从外界通过 access 端口时，数据本身是不带 tag 的。access 端口给数据打上 tag. 当数据从 access 端口发送时，vid 必须与 端口的 vid 一致，否则丢弃数据，当数据经过 access 端口发出时，会先将帧的 tag 信息去掉再发送。

2. trunk 端口： 声明一组 vid ， 只允许带这些 vid 的数据帧通过，用于交换机之间通信，数据进出都带 tag

#### 单主机 vlan 划分

	docker run -itd --name ovs-test-01 --network none ubuntu:16.04 
	docker run -itd --name ovs-test-02 --network none ubuntu:16.04 
	docker run -itd --name ovs-test-03 --network none ubuntu:16.04 
	docker run -itd --name ovs-test-04 --network none ubuntu:16.04 

	pipework ovs0 ovs-test-01 172.20.0.2/24 @100
	pipework ovs0 ovs-test-02 172.20.0.3/24 @100
	pipework ovs0 ovs-test-03 172.20.0.4/24 @200
	pipework ovs0 ovs-test-04 172.20.0.5/24 @200
	
	然后互相 ping 一下试试

	可以查看创建的网桥 ovs-vsctl show
	Bridge "ovs0"
		Port "veth1pl1144"
			tag: 100
			Interface "veth1pl1144"
		Port "veth1pl1226"
			tag: 200
			Interface "veth1pl1226"
		Port "veth1pl1261"
			tag: 200
			Interface "veth1pl1261"
		Port "ovs0"
			Interface "ovs0"
				type: internal
		Port "veth1pl1187"
			tag: 100
			Interface "veth1pl1187"
					
#### 多主机划分 vlan 
将主机的一个网卡加入到 ovs 交换机中，设为 trunk 端口。

	ip link set eth1 promisc on
	ovs-vsctl add-port ovs0 eth1

ps: 这种方式我在 vmware 上用两台虚拟机搭建，没有成功，没有找到原因在哪里。可能是网卡的问题？

### Overly 隧道模式
即将一种协议包装在另一种协议中传输的技术，当前的 Overly 技术主要有 Vxlan 和 NVGRE 技术

步骤

	1. 设置 DOCKER_OPTS="--fixed-cidr=172.17.2.0/24"  在 /etc/default/docker 文件中
	2. 有个 bug 导致这个配置不起作用，修改 /lib/systemd/system/docker.service
		EnvironmentFile=-/etc/default/docker
		ExecStart=/usr/bin/dockerd $DOCKER_OPTS -H fd://
	3. 配置网络
		ovs-vsctl add-br ovs0
		brctl addif docker0 ovs0
		ovs-vsctl add-port ovs0 gre0 -- set interface gre0 type=gre options:remote_ip=192.168.192.133
	
	另一台主机相同

ps: 这种方式我也没有成功，同样的问题，跨主机包就不知道跑哪里去了。

### Dockerfile 最佳实践
	RUN 命令
		推荐 RUN ["executable","param","param"] 写法
	CMD 命令
		推荐 CMD ["executable","param","param"] 写法
		如果指定多条，只有最后一条会起作用。如果用户在创建容器时指定了命令，会覆盖掉 CMD 指定的命令
	ENTRYPOINT 
		推荐 ENTRYPOINT ["executable","param","param"] 写法	
		run 命令指定的参数能覆盖掉 CMD 但是不能覆盖 ENTRYPOINT
		如果使用 ENTRYPOINT "command" 这种shell格式，程序运行在 bin/sh -c 中, 不能接收信号
	EXPOSE 
		不要在 dockerfile 中做端口映射，可以仅仅暴漏端口
		可以写作 EXPOSE 80  不要写作 EXPOSE 80:8080

	
### 容器监控

	docker stats 显示 cpu 内存 信息 
	docker top 查看容器中进程运行的情况

### 常用的监控软件

	cAdvisor,Datadog


	
	
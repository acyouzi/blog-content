---
title: docker-basis
date: 2017-01-12 23:26:14
author: "acyouzi"
# cdn: header-off
# header-img: img/docker.png
tags:
	- docker
	- 容器
---

### docker 基础命令

docker network create nw-link-test
docker run -it --name link-test-01 --network nw-link-test  --link link-test-02:link-test-02 ubuntu-apt-163:16.04 /bin/bash
docker run -it --name link-test-02 --network nw-link-test  --link link-test-01:link-test-01 ubuntu-apt-163:16.04 /bin/bash


namespace 做资源隔离

uts 
ipc
pid
network
mount
user

cgroup 做资源限制

联合挂载

	分层
	写时复制
	内容寻址
	联合挂载

Docker 镜像组织方式

	可读写部分
	init-layer
	只读层

镜像存在
/var/lib/docker/image/aufs

Docker 存储驱动

	aufs
		支持联合挂载
			支持将不同目录挂载到同一个目录上，这个过程对用户是透明的。一般可读写层在上，下层是只读层。
		/var/lib/docker/image/aufs 用于存放镜像相关元数据
		/var/lib/docker/aufs 下边有 diff layers mnt 三个目录
			/mnt 为 aufs 挂载目录
			/diff 为实际数据来源，包括只读层和可读可写层
			/layers 为每层依赖的层描述文件 就是这个层依赖哪些层
		
			/<mountID>-init 层作为最后一层只读层，用于挂载并重新生成一下文件
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

				docker ps 查看短ID
				cd /var/lib/docker/ 
				cat 查看 mountID
				cat image/aufs/layerdb/mounts/2313d627f4df7b7cec52cd8eb1cd9c628fedfa8b93cbf5095de87d3d9958fc7a/mount-id
				得到 mountID 为 4a6f315dfb32c50e7ff54589d52f0f93acf1a74fa50c381bb35d54c6bbb41e3e
				然后找到镜像挂载位置
				/var/lib/docker/aufs/mnt/4a6f315dfb32c50e7ff54589d52f0f93acf1a74fa50c381bb35d54c6bbb41e3e 

				du --max-depth=1 -h ./aufs/mnt | grep 4a6f315dfb3 查看大小， 在容器启动的状态下挂载文件

				进入容器，在root 目录下创建一个文件，然后关闭容器
				进入命令 docker exec -it test /bin/bash
				停止后查看镜像挂载目录
				发现
					/var/lib/docker/aufs/mnt/4a6f315dfb32c50e7ff54589d52f0f93acf1a74fa50c381bb35d54c6bbb41e3e/  目录空了
					/var/lib/docker/aufs/diff/4a6f315dfb32c50e7ff54589d52f0f93acf1a74fa50c381bb35d54c6bbb41e3e/ 下有一个 root/ 目录，目录下边是我们修改的文件

				这就是联合挂载的效果

				然后再次启动容器，进入容器，修改 /etc/hosts 文件。
				然后进入 /var/lib/docker/aufs/diff/4a6f315dfb32c50e7ff54589d52f0f93acf1a74fa50c381bb35d54c6bbb41e3e/ 
				发现 这里并没有 etc/hosts 文件
				然后进入 /var/lib/docker/aufs/diff/4a6f315dfb32c50e7ff54589d52f0f93acf1a74fa50c381bb35d54c6bbb41e3e-init/
				发现存在 etc/hosts 文件，但是没有任何内容。
				而且重启容器之后，修改被还原了。

				这是 init 层的效果
	
	devicemapper
	vfs
	overlay
		新型联合文件系统，理论上性能好于 aufs


数据卷 volume

docker volume 命令
	create      Create a volume
	inspect     Display detailed information on one or more volumes
	ls          List volumes
	rm          Remove one or more volumes

-v 挂载数据卷
	-v /xxx 自动创建一个 volume 并挂载到容器中
	-v volume_name:/xxx 挂载一个已经存在的 volume 
	-v /host/dir:/xxx 挂载一个本地目录到容器
	一个容器可挂载多个 volume 

	dockerfile 中挂载 volume 不能指定要挂载的文件。为了可移植性，只能自动创建
	共享 volume
		--volume-from container-xx  挂载 container-xx 挂载的全部 volume


容器网络
	libnetwork 支持5种内置驱动
		bridge 驱动
		host 驱动 
			这种模式下容器没有网络协议栈，没有独立的 network namespace
		overlay 驱动 
			使用 vxlan 方式，适合 SDN ， 需要额外配置一个存储服务，如 etcd zookeeper
		remote驱动
			调用用户自己实现的网络驱动的插件
		null 驱动
			这种模式下有自己的 network namespace, 但是没有进行任何网络配置

	使用 docker network 管理网络
		connect     Connect a container to a network
		create      Create a network
			可以指定驱动类型，默认是 bridge 
			创建完，通过 brclt show 可以发现多了一个网桥
		disconnect  Disconnect a container from a network
		inspect     Display detailed information on one or more networks
		ls          List networks
		rm          Remove one or more networks

	
	network 测试
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
	
	route 规则
		在宿主机上通过 route 命令查看路由表
			route -n 

			172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
			172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-91694b4f37a8
			172.19.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-63fb8f706b58
			172.20.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-e63a8cb0655b

		route 命令使用
			直接在命令行下执行route命令来添加路由，不会永久保存，当网卡重启或者机器重启之后，该路由就失效了；
			可以在/etc/rc.local中添加route命令来保证该路由设置永久有效
		
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

		docker daemon 启动时，可以指定的网络配置, 修改 /etc/default/docker 文件的 DOCKER_OPTS
			--bip=172.17.0.1/24 设置 docker0 的 ip 地址和子网范围
			--fixed-cidr=172.17.0.1/25 设置容器的 ip 获取范围，默认是 docker0 的子网范围
       		--mtu=BYTES 

		也可以自己创建网桥然后在 daemon 启动的时候，使用 --bridge=xxx 来使用指定网桥

			brclt addbr br0
			ifconfig br0 xxx.xxx.xxx.xxx

		也可以跟宿主机的物理网卡相连接
			
			brclt addif br0 ens33

	iptable 规则 
		使用 iptables-save 命令打印 iptables rules . iptables-save — dump iptables rules to stdout

		容器与外部通信
			-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
			从源地址 172.17.0.0/16 发出，但不是从 docker0 网卡发出的做 snat (源地址转换)

			另外涉及多个网卡之间信息的转发，需要设置 ip_forward 参数为 1
			echo 1 > /proc/sys/net/ipv4/ip_forward
		容器之间通信
			-A FORWARD -i docker0 -o docker0 -j ACCEPT

		相关参数 
			--iptables=true 是否允许修改宿主机的 iptable
			--icc=true 是否允许容器间相互通信
			--ip-forward=true 是否开启 ip_forward 功能
			-h hostname   "/etc/hostname"
			--dns=ipAddr DNS 服务器 "/etc/resolv.conf"

	磁盘限额

		--store-opt 选项只对 devicemapper 文件系统有效
		可以通过创建虚拟文件系统来对磁盘进行限额

		创建文件
		dd if=/dev/zero of=./disk-quota.ext4 count=2048 bs=1MB
		格式化
		mkfs -t ext4 -F  disk-quota.ext4
		挂载
		mount -o loop,rw,usrquota,grpquota ./disk-quota.ext4 ./test/

	流量限制

		trafic controller 

容器化思维
	容器就是一个进程，只不过拥有相对独立的环境，所以不要去备份容器，不要去在容器里面开启 ssh

network namespace
	ip 命令操作 namespace
	ip netns
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

	ip netns exec ns-test /bin/bash 可以进入到 namespace 里面执行命令，使用 exit 退出

	ip 命令配置网卡
		ip link set dev lo up 启动设备，在新创建的 network namespace 里面有一个 lo 设备没有启动，最好手动启动

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

		使用 pipework 可以更简单的完成网络配置

	pipework
		简单使用 
			pipework br0 container-name 172.17.2.24/24@172.17.2.1
			@ 后是网关

		macvlan 链接本地网络
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
				
			支持dhcp 
				pipework eth0 container-name dhcp

		Open vSwitch
			Open vSwitch 是一个开源的虚拟交换机，创建 Open vSwitch 网桥需要网桥名以 ovs 开头
			
			ovs-vsctl add-br "$IFNAME"
			ovs-vsctl add-port "$IFNAME" "$LOCAL_IFNAME" ${VLAN:+tag="$VLAN"}

		支持 vlan 
			只有 Open vSwitch 和 macvlan 支持划分 vlan，普通 linux 网桥不可

			pipework osv-br0 container dhcp @12
			@ 后面是 vlan 号
		
		




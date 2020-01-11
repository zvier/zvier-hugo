---
title: "Linux Network Namespace"
date: 2019-11-18T06:07:29+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
积羽沉舟，群轻折轴，众口铄金，积毁销骨。《史记·张仪列传》
<!--more-->
# linux 名字空间简述
linux namespace技术是实现虚拟化的基础，它的作用就是隔离各种内核资源。根据所隔离资源的不同，namespace目前有如下6种:   
1. Mount namespace 用于隔离文件系统的挂载点  
2. Network namespace 用于隔离linux系统的网络资源  
3. IPC namespace 用于隔离POSIX进程间通信的消息队列  
4. PID namespace 用于隔离进程PID数字空间  
5. UTS namespace 用于隔离主机名  
6. User namespace 用于隔离用户权限  

linux通过名字空间隔离的方式，将进程关进一个小房间，但房间虽小，五脏俱全，以便进程在这里也可以生活得很好。   

# network namespace基本操作
如上所述，linux network
namespace可用于隔离linux系统的各种网络资源，包括网络设备，比如: IP地址，端口，路由，以及/proc/net目录等，使得进程用于自己的网络空间。使用ip工具的netns子命令可以轻松管理linux的network namespace：  

* 创建network namespace(增)
{{< highlight bash "linenos=inline" >}}
ip netns add netns-name
{{< /highlight >}}
创建完后，可以看到在目录/var/run/netns下会生一个同名文件目录/var/run/netns/netns1，作为该network namespace的一个挂载点，方便对其进行管理，即便没有进程也可以继续存在  

* 进入network namespace执行命令(改)  
进入network namespace后，可以对该命名空间进行很多操作，包括配置网络  
{{< highlight bash "linenos=inline" >}}
ip netns exec netn-name ip link list
{{< /highlight >}}
这条命令和docker exec命令是不是很像?
{{< highlight bash "linenos=inline" >}}
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
{{< /highlight >}}
可见，命名空间初始创建后，自带一个lo设备，不过当前还是DOWN状态，此时进入到命名空间去ping这个设备，其实是不可达的。
{{< highlight bash "linenos=inline" >}}
ip netns exec netns-name ping 127.0.0.1
connect: Network is unreachable
{{< /highlight >}}
接下来，我们把这个回环设备用ip link命令启起来，然后再ping该设备
{{< highlight bash "linenos=inline" >}}
ip netns exec netns-name ip link set dev lo up
{{< /highlight >}}
{{< highlight bash "linenos=inline" >}}
ip netns exec netns-name ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.039 ms
{{< /highlight >}}

* 列出命名空间(列)
{{< highlight bash "linenos=inline" >}}
ip netns list
{{< /highlight >}}

* 删除命名空间(删）
{{< highlight bash "linenos=inline" >}}
ip netns delete netns-name
{{< /highlight >}}

# 配置network namespace
为了于外界网络通信，network namespace还需要有虚拟以太网卡，即veth pair对，报文从veth pair一端进，另一端出，就像一根连接两个空间的管道。以下为如何创建和设置veth pair，以便两个不同网络命名空间可以通过veth pair通信。    
1. 利用ip link命令创建虚拟以太网卡veth0以及它的对端veth1
{{< highlight bash "linenos=inline" >}}
ip link add veth0 type veth peer name veth1
{{< /highlight >}}
2. 放置veth pair  
创建完后，这对veth pair初始都在根network namespace中
{{< highlight bash "linenos=inline" >}}
ip a
4: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether 9e:a3:a2:62:65:96 brd ff:ff:ff:ff:ff:ff
5: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether 26:01:b6:8b:8e:8e brd ff:ff:ff:ff:ff:ff
{{< /highlight >}}
为了达到让两个不同命名空间可进行网络通信的目的，其中一端必须借助ip link命令搁置到另一个网络命名空间
{{< highlight bash "linenos=inline" >}}
ip link set veth1 netns net-name
{{< /highlight >}}
此时，在根network namespace上用ip a命令查看这对veth pair，发现veth1不见了，只剩veth0。可以说veth pair就像一根绳子，上面的操作无非就是把绳子的一头抛向另一个空间
{{< highlight bash "linenos=inline" >}}
5: veth0@if4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether 26:01:b6:8b:8e:8e brd ff:ff:ff:ff:ff:ff link-netnsid 0
{{< /highlight >}}
再跑到netns-name中看以下，可以发现veth1已经到netns-name中来来
{{< highlight bash "linenos=inline" >}}
ip netns exec netns-name ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
4: veth1@if5: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether 9e:a3:a2:62:65:96 brd ff:ff:ff:ff:ff:ff link-netnsid 0
{{< /highlight >}}
3. 绑定ip启动网卡  
光将veth1放置到netns-name中，两个空间还是不能通信的。还需要用ifconfig命令为分别为两个网卡绑定ip并启动
{{< highlight bash "linenos=inline" >}}
ifconfig veth0 10.0.0.1/24 up
ip netns exec netn-name ifconfig veth1 10.0.0.2/24 up
{{< /highlight >}}
4. 验证  
现在，在主机上ping命名空间netns-name的网卡veth1地址，或者在命名空间netns-name内ping主机上的veth0的地址
{{< highlight bash "linenos=inline" >}}
ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.048 ms

ip netns exec netns-name ping 10.0.0.1
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.047 ms
{{< /highlight >}}


# 尝试连接到因特网
上面我们通过创建veth pair对，将两个不同的空间连接起来，但这个新创建的网络命名空间netns-name与现实中的因特网还是隔离的，我们可以用route以及iptables命令查看下命名空间中路由以及iptables规则，发现空空如也，几乎什么也没有，所以它与因特网目前还是完全隔离的  
{{< highlight bash "linenos=inline" >}}
ip netns exec netns-name route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 veth1
{{< /highlight >}}
{{< highlight bash "linenos=inline" >}}
ip netns exec netns-name iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
{{< /highlight >}}
为了让这种新创建的网络命名空间也能与因特网通信，有很多方法，典型的就是在主机network namespace上创建一个linux网桥，并将veth0附在其上，或者使用NAT规则将报文转发出去，关于这些技术后面再细研究。

值得注意的是：虚拟网络设备可以随意分配到根网络命名空间或者自创建的网络命名空间，而真实的物理硬件网络设备则只能放置在根网络命名空间中。  

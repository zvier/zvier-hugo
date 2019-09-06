---
title: "iptables端口转发常用操作"
date: 2019-07-10T20:36:19+08:00
draft: true
categories: [""]
tags: ["iptables"]
---
# 背景
最近经常操作iptables，以及遇到由于iptables导致的问题，遂总结下常用操作  

# 准备工作
默认情况下，网卡只允许INPUT和OUTPUT流量，欲使用iptables做一些流量转发操作，必须先开启网卡转发功能，使得FORWARD流量可以被转发    
{{< highlight bash "linenos=inline" >}}
sysctl net.ipv4.ip_forward=1
sysctl -a | grep 'ip_forward'
{{< /highlight >}}
当然也可以手动配置   
{{< highlight bash "linenos=inline" >}}
echo '1' > /proc/sys/net/ipv4/ip_forward
sysctl -p                         # 生效
{{< /highlight >}}

# iptables 规则链

|规则链|功能|
|:---|:----|
|INPUT|处理输入数据包|
|OUTPUT|处理输出数据包|
|FORWARD|处理转发数据包|
|PREROUTING|DNAT|
|POSTOUTING|SNAT|

# 本机端口转发
{{< highlight bash "linenos=inline" >}}
iptables -t nat -A PREROUTING -p tcp --dport ${port} -j REDIRECT --to-ports ${dst-port}
iptblaes -t nat -A PREROUTING -p udp --dport ${port} -j REDIRECT --to-ports ${dst-port}
{{< /highlight >}}

## iptables表
上面-t选项用于指定表，默认为filter表，iptables表有如下

|表名|功能|
|:---|:----|
|filter|包过滤，用于防火墙规则|
|nat|地址转换，用于网关路由器|
|raw|高级功能，如：网址过滤|
|mangle|数据包修改（QOS），用于实现服务质量|

## 动作
|动作|功能|
|:---|:----|
|ACCEPT|接收数据包|
|DROP|丢弃数据包|
|REDIRECT|重定向、映射、透明代理|
|SNAT|源地址转换|
|DNAT|目标地址转换|
|MASQUERADE|IP伪装（NAT），用于ADSL|
|LOG|日志记录|

其它选项  
* -p 协议   
* --dport 目标端口   
* --to-port port 常配合REDIRECT使用，端口重定向  


## 设置默认策略
-P,--policy 设置默认策略，也就是未符合过滤条件之封包，预设的处理方式，默认策略有ACCEPT和DROP两种方式   
{{< highlight bash "linenos=inline" >}}
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP
{{< /highlight >}}
-P 为规则链设置默认策略

# 查看
-L,--list：显示规则链中已有的条目
{{< highlight bash "linenos=inline" >}}
iptables -t nat -nL --line-numbers
iptables -L INPUT
{{< /highlight >}}


# 删除
-D,--delete：从规则链中删除条目
{{< highlight bash "linenos=inline" >}}
iptbles -t nat -D PREROUTING 1
iptables -D INPUT 1
{{< /highlight >}}


# 清除所有策略
-F,--flush 删除某规则链中的所有规则  
{{< highlight bash "linenos=inline" >}}
iptables -F
iptables --flush
iptables -F INPUT
{{< /highlight >}}


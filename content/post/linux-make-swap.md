---
title: "linux make swap"
date: 2020-01-01T14:45:46+08:00
draft: true
categories: ["技术"]
tags: ["linux"]
---
<!--more-->
# 问题
编译containerd时，提示错误信息:
{{< highlight bash "linenos=inline" >}}
linux_amd64/link: signal: killed
{{< /highlight >}}
原因是服务器内存不够，可以通过创建交换分区解决

# 方式
1. 创建要作为swap分区的文件，增加1GB大小的交换分区，文件大小=bs*count
{{< highlight go "linenos=inline" >}}
dd if=/dev/zero of=/root/swapfile bs=1M count=1024
{{< /highlight >}}
dd命令从标准输入或文件中读取指定大小的块数据，输出到文件、设备或标准输出，在拷贝的同时进行指定转化，convert and copy a file  
dd命令参数介绍:  
if=输入文件(或设备名称)  
of=输出文件(或设备名称)
bs=bytes 同时设置读/写缓冲区的字节数  
count=blocks 只拷贝输入的blocks块   
上述命令的意思就是创建一个1G大小的空文件，内容全0，其中/dev/zero是一个字符输入设备，可用来初始化文件，该设备可以无穷尽地提供0值字节，可以使用任何你需要的数目，通常用于向设备或文件写入字符串0。

2. 格式化交换分区文件，建立swap文件系统
{{< highlight go "linenos=inline" >}}
mkswap /root/swapfile
{{< /highlight >}}
mkswap用于将磁盘分区或文件设为Linux的交换区(swap area)

3. 启动交换分区文件
{{< highlight go "linenos=inline" >}}
swapon /root/swapfile
{{< /highlight >}}
swapon命令用于激活Linux系统中交换空间，Linux系统的内存管理必须使用交换区来建立虚拟内存，相对还有一个关闭swap的命令swapoff

4. 配置开启启动，在文件/etc/fstab中添加如下行
{{< highlight go "linenos=inline" >}}
/root/swapfile swap swap defaults 0 0
{{< /highlight >}}

---
title: "runc usage"
date: 2020-03-21T15:04:42+08:00
draft: true
categories: ["技术"]
tags: ["runc"]
---
我亦飘零久，十年来，深恩负尽，死生师友。——顾贞观《金缕曲二首》
<!--more-->
# runc和OCI
runc是一个根据OCI(Open Container
Initiative)标准来创建和运行容器的轻量级工具，它可以不通过docker引擎，就能创建、运行和销毁容器。

> OCI: An open governance structure for the express purpose of creating open industry standards around container formats and runtime.

OCI目前主要有两个标准文档: 容器运行时标准(runtime spec)和容器镜像标准(image spec)，这两个标准通过OCI runtime filesytem bundle的标准格式连接在一起，OCI镜像可以通过工具转换成bundle，OCI容器引擎比如这里的runc，能够识别和根据这个bundle来运行容器。


# 使用
## 准备OCI bundle
runc是运行容器的运行时，它负责利用符合标准的文件等资源运行容器，但是它不包含docker那样的镜像管理功能。如果要用runc运行容器，先得准备好容器的文件系统以及配置文件。  

所谓的OCI bundle就是指容器的文件系统和配置文件config.json。有了容器的文件系统后，可以通过runc spec 命令来生成config.json文件。

以下使用docker export生成容器的文件系统，注意这里必须解压到rootfs目录

{{< highlight go "linenos=inline" >}}
docker export $(docker create busybox) | tar -C rootfs -xvf -
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
ls rootfs/
bin  dev  etc  home  proc  root  sys  tmp  usr  var
{{< /highlight >}}

现在rootfs目录下就是busybox镜像的文件系统，然后再同级目录下，执行runc spec生成config.json文件
{{< highlight go "linenos=inline" >}}
runc spec

ls
config.json  rootfs
{{< /highlight >}}

适当修改下config.json文件，把"terminal": true改为false，把"args": ["sh"] 改为 "args": ["sleep", "30"]，方便调试

# 容器状态
OCI标准都定义了容器状态5种不同状态：

1.creating：使用create命令创建容器，这时容器正在创建中。  
2.created：容器已经创建出来，但是还没有运行，表示镜像文件和配置没有错误，容器能够在当前平台上运行。   
3.running：容器里面的进程处于运行状态，正在执行用户设定的任务。   
4.stopped：容器运行完成，或者运行出错，或者stop命令之后，容器处于暂停状态。这个状态，容器还有很多信息保存在平台中，并没有完全被删除。  
5.paused：暂停容器中的所有进程，可以使用resume命令恢复这些进程的执行。   

# runc常用命令
## 辅助命令
{{< highlight go "linenos=inline" >}}
runc -v
runc help subcommand
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
runc create busybox
runc list
runc state busybox
{{< /highlight >}}
通过create成功创建了容器后，容器的状态就是"created"。

## 查看容器内进程
使用ps命令可以查看容器内运行的进程
{{< highlight go "linenos=inline" >}}
runc ps busybox

UID        PID  PPID  C STIME TTY          TIME CMD
root     16515     1  0 15:47 ?        00:00:00 runc init
{{< /highlight >}}
可以看到，此时busybox容器内有一个名为init的进程在运行。

使用 start 命令执行容器中定义的任务
{{< highlight go "linenos=inline" >}}
runc start
{{< /highlight >}}
使用start 命令启动容器后，让我们再用 ps 命令看看容器内运行了什么进程：
{{< highlight go "linenos=inline" >}}
runc ps busybox

UID        PID  PPID  C STIME TTY          TIME CMD
root     16515     1  0 15:47 ?        00:00:00 sleep 30
{{< /highlight >}}

此时我们在 config.json 中定义的 sleep 进程在运行。再用 state 命令看看容器此时的状态，此时已经变成了 running！
{{< highlight go "linenos=inline" >}}
runc state busybox

{
  "ociVersion": "1.0.1-dev",
  "id": "busybox",
  "pid": 0,
  "status": "stopped",
  "bundle": "/home/zvier/test/busybox",
  "rootfs": "/home/zvier/test/busybox/rootfs",
  "created": "2020-03-21T07:47:25.729499324Z",
  "owner": ""
}
{{< /highlight >}}

## 容器中执行命令
使用exec命令，可以在处于created和running状态的容器中执行命令：

{{< highlight go "linenos=inline" >}}
runc exec busybox ls
{{< /highlight >}}
当容器中的用户任务结束后，容器会变成 stopped 状态，这时就不能再通过 exec 执行其它的命令了。

## 删除容器
通过delete命令可以删除容器，一般情况下是删除stopped状态的容器
{{< highlight go "linenos=inline" >}}
runc delete mybusybox
{{< /highlight >}}

使用 run 命令创建并运行容器
就像 docker run 命令一样，它会创建容器并运行容器中的命令：

{{< highlight go "linenos=inline" >}}
runc run mybusybox
{{< /highlight >}}

当容器中的命令退出后容器随即被删除。

## 停止容器中的任务
使用kill命令可以停止容器中正在运行的任务：

{{< highlight go "linenos=inline" >}}
runc kill busybox
{{< /highlight >}}

kill默认会优雅的结束容器中的进程，遇到特殊情况，可以加参数9
{{< highlight go "linenos=inline" >}}
runc kill busybox 9
{{< /highlight >}}

## 暂停容器中的进程  
使用pause命令可以暂停容器中的所有进程
{{< highlight go "linenos=inline" >}}
runc pause busybox
{{< /highlight >}}

执行pause命令后，容器的状态由running变成了paused。然后我们再通过resume命令恢复容器中进程的执行
{{< highlight go "linenos=inline" >}}
runc resume mybusybox
{{< /highlight >}}

此时容器的状态又恢复到了running。

## 查看容器中的资源使用情况
使用events命令可以获取容器的资源使用情况，events命令能够报告容器事件及其资源占用的统计信息
{{< highlight go "linenos=inline" >}}
runc events busybox
{{< /highlight >}}

rootless containers
前面我们运行的所有命令都是以 root 权限执行的。能不能以普通用户的权限运行容器呢？答案是可以的，并被称为 rootless。要想以 rootless 的方式运行容器，需要我们在生成容器的配置文件时就为 spec 命令指定 rootless 参数：

$ runc spec --rootless
并且在运行容器时通过 --root 参数指定一个存放容器状态的路径：

$ runc --root /tmp/runc run mybusybox

容器的热迁移操作
RunC 支持容器的热迁移操作，所谓热迁移就是将一个容器进行 checkpoint 操作，并获得一系列文件，使用这一系列文件可以在本机或者其他主机上进行容器的 restore 工作。这也是 checkpoint  和 restore 两个命令存在的原因。热迁移属于比较复杂的操作，目前 runC 使用了 CRIU 作为热迁移的工具。RunC 主要是调用 CRIU（Checkpoint and Restore in Userspace）来完成热迁移操作。CIRU 负责冻结进程，并将作为一系列文件存储在硬盘上。并负责使用这些文件还原这个被冻结的进程。

# 总结
runc作为标准化容器运行时的一个实现，已经被docker内置为默认的容器运行时。随着runc自身的成熟和完善会有越来越多的大厂把runc作为默认的容器运行时。


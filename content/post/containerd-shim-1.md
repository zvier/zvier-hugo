---
title: "containerd shim详解 (一)"
date: 2019-06-21T20:45:35+08:00
draft: true
categories: ["docker"]
tags: [""]
---
# containerd-shim是什么
来吧，让我先来撸一遍社区的containerd-shim。  

shim直白翻译过来就是垫片的意思，如下图螺栓与螺母之间的小铁片就是垫片，用于紧固联接件、预防零件变形、滑丝受损等。

![containerd-shim](/img/containerd-shim/shim.jpg)

那容器垫片containerd-shim又是什么鬼呢？当我们ps一下docker宿主机上的进程时，可以看到很多containerd-shim的进程：
<!--more-->

{{< highlight shell "linenos=inline" >}}
ps -ef | grep  containerd-shim | grep -v grep
root       661   807  0 May01 ?        00:02:51 containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/45c8daa9271ad3305089b0820556069480417596a1a2acf01ce26eaf64a74ae0 -address /run/containerd/containerd.sock -containerd-binary /usr/bin/containerd -runtime-root /var/run/docker/runtime-runc
root      2583   807  0 Apr13 ?        00:03:57 containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/8e0ccb321e59581899ecfc72a19466340bb3971b68f041e34ad84a3363526633 -address /run/containerd/containerd.sock -containerd-binary /usr/bin/containerd -runtime-root /var/run/docker/runtime-runc
root      2584   807  0 Apr13 ?        00:04:29 containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/cd01d511606d2bb66e8b710f261aa894b044ab31439ab7bdad7fdffe1ce08379 -address /run/containerd/containerd.sock -containerd-binary /usr/bin/containerd -runtime-root /var/run/docker/runtime-runc
{{< /highlight >}}
可以猜想每个容器都有这样一个垫片，粗略验证一下
{{< highlight shell "linenos=inline" >}}
ps -ef | grep  containerd-shim | grep -v grep | wc -l
29
docker ps | wc -l
29
docker ps -a | wc -l
36
{{< /highlight >}}

> 每个running状态的容器都有一个containerd-shim

# containerd-shim在容器架构中的位置
既然每个running状态的容器都有一个containerd-shim，那我们再来追究下它们之间的关系
{{< highlight shell "linenos=inline" >}}
docker top 02698370aa4b
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
27                  13831               13813               0                   Jun20               ?                   00:24:59            /mysql-agent --v=4
ps -ef | grep 13813 | grep -v grep
root     13813   807  0 Jun20 ?        00:02:15 containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/02698370aa4b483569f02b7fa7e87f65c7b630b662cf3c72b60ebdfc8b8a599a -address /run/containerd/containerd.sock -containerd-binary /usr/bin/containerd -runtime-root /var/run/docker/runtime-runc
27       13831 13813  0 Jun20 ?        00:24:59 /mysql-agent --v=4

ps -ef | grep /usr/bin/containerd | grep -v containerd-shim | grep -v grep
root       807     1  0 Apr13 ?        01:51:02 /usr/bin/containerd
{{< /highlight >}}
可见容器中进程的父进程就是containerd-shim，而containerd-shim的父进程正是containerd。

> docker top ${cid}
显示的PID为容器内进程在宿主机上的PID，PPID为容器内进程在宿主机上父进程的PID。

再来pstree一下containerd的pid，可以观察到类似如下进程关系
{{< highlight shell "linenos=inline" >}}
pstree 807
containerd─┬─containerd-shim─┬─mysqld───29*[{mysqld}]
           │                 └─10*[{containerd-shim}]
           ├─8*[containerd-shim─┬─pause]
           │                    └─9*[{containerd-shim}]]
           ├─containerd-shim─┬─flanneld───7*[{flanneld}]
           │                 └─9*[{containerd-shim}]
           ├─containerd-shim─┬─kube-proxy───6*[{kube-proxy}]
           │                 └─9*[{containerd-shim}]
           ├─containerd-shim─┬─traefik───7*[{traefik}]
           │                 └─10*[{containerd-shim}]
           ├─containerd-shim─┬─coredns───7*[{coredns}]
           │                 └─9*[{containerd-shim}]
           └─32*[{containerd}]
{{< /highlight >}}


containerd-shim作为一个shim，位于containerd与runc之间，是容器的父进程
![containerd-shim](/img/containerd-shim/docker-shim.png)

通常创建容器时的流程为   
![docker-shim](/img/containerd-shim/request-flow.png)
可以看出，containerd在收到容器引擎的rpc请求后，并不会直接操作容器，而是通过创建了一个containerd-shim进程，通过containerd-shim调用runc命令行来操纵容器。  

# containerd-shim的作用
containerd-shim的存在解耦了containerd和真实容器，那这种解耦又有什么好处呢？  

1. 如果containerd挂了或者正在升级，只要containerd-shim正常，就可以确保容器打开的文件描述符不会关掉  
2. 当容器生命周期结束时，我们可以依靠containerd-shim来替容器收尸，避免产生僵尸进程  
3. containerd-shim内存占用更小，有了containerd-shim，可以允许runc在创建运行容器之后退出。runc是一个go实现编译出来的二进制文件，默认是静态链接，如果宿主机每启动一个容器都消耗一个runc，那随着容器数的增加，内存消耗会快速线性增长。


# 总结
1. containerd-shim是containerd的子进程，每个running状态的容器都有一个   
2. 允许runc在创建, 运行容器之后, runc退出，节省内存开销  
3. 用shim作为容器的父进程，而不是直接用containerd作为容器的父进程，是为了防止这种情况：当containerd挂掉的时候，shim还在，因此可以保证容器打开的文件描述符不会被关掉
4. 依靠shim来收集&报告容器的退出状态，这样就不需要containerd来wait子进程   

# 参考
[深入理解Docker容器技术](https://imfox.io/2018/10/02/deep-in-docker/)   
[Docker组件介绍（二）：shim, docker-init和docker-proxy](https://jiajunhuang.com/articles/2018_12_24-docker_components_part2.md.html)   
[Use of containerd-shim in docker-architecture](https://groups.google.com/forum/#!topic/docker-dev/zaZFlvIx1_k)   
[白话 Kubernetes Runtime](https://juejin.im/entry/5c8e5c28e51d4554ad53a1fc)   

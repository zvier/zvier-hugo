---
title: "overlay2"
date: 2019-11-26T21:55:57+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
林断山明竹隐墙，乱蝉衰草小池塘。翻空白鸟时时见，照水红蕖细细香。
村舍外，古城旁，杖藜徐步转斜阳。殷勤昨夜三更雨，又得浮生一日凉。
——苏轼《鹧鸪天》
<!--more-->
# 实验
拉取一个ubuntu镜像
{{< highlight go "linenos=inline" >}}
docker pull ubuntu

Using default tag: latest
latest: Pulling from library/ubuntu
7ddbc47eeb70: Pull complete
c1bbdc448b72: Pull complete
8c3b70e39044: Pull complete
45d437916d57: Pull complete
Digest: sha256:6e9f67fa63b0323e9a1e587fd71c561ba48a034504fb804fd26fd8800039835d
Status: Downloaded newer image for ubuntu:latest
{{< /highlight >}}
可以看到，该ubuntu镜像有4层，这4层都是只读层，查看一下存储的目录/var/lib/docker/overlay2/，可以分别看到镜像层对应的目录
{{< highlight go "linenos=inline" >}}
ls /var/lib/docker/overlay2/
06e334c99ed7523d23207331c69f33cae27ba27cba6aca792123a18294b7ff54  9266b15e0526e00dec39de79579edd308a6929851aa4cd6f1abdbee24086005b  l
3ce856fe582ee3a07fc83315f11506baf4ec156dda2b3b2f2aef425d83578d2f  a12e3822e217c5e8cb1f47533a9c845a15004ac239dd7e6fefd39d70ab6059ca
{{< /highlight >}}
除此之外，还有一个l目录
{{< highlight go "linenos=inline" >}}
ll /var/lib/docker/overlay2/l
total 16
lrwxrwxrwx 1 root root 72 Nov 26 21:58 5SCDQNUHYPUCLUEFYTFLJXRYIB -> ../06e334c99ed7523d23207331c69f33cae27ba27cba6aca792123a18294b7ff54/diff
lrwxrwxrwx 1 root root 72 Nov 26 21:59 MIA5PW7ES7CROKZ2IQKRRCZZ4M -> ../a12e3822e217c5e8cb1f47533a9c845a15004ac239dd7e6fefd39d70ab6059ca/diff
lrwxrwxrwx 1 root root 72 Nov 26 21:59 N6GWFEGZOEYSV66LU2SWBHZ54T -> ../3ce856fe582ee3a07fc83315f11506baf4ec156dda2b3b2f2aef425d83578d2f/diff
lrwxrwxrwx 1 root root 72 Nov 26 21:59 XTSVQXPU33EB6U37TGSTZD6ERF -> ../9266b15e0526e00dec39de79579edd308a6929851aa4cd6f1abdbee24086005b/diff
{{< /highlight >}}
可以看到，l目录下都是4个到各层diff目录的软连接。继续进入分别查看下这些目录的内容
{{< highlight go "linenos=inline" >}}
ls 5SCDQNUHYPUCLUEFYTFLJXRYIB
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

ls MIA5PW7ES7CROKZ2IQKRRCZZ4M
etc  sbin  usr  var

ls N6GWFEGZOEYSV66LU2SWBHZ54T
run

ls XTSVQXPU33EB6U37TGSTZD6ERF
var
{{< /highlight >}}
从内容来看，每层的diff就是文件系统在统一挂载时的挂载点，通过多层挂载最终形成一个统一可用的文件系统。由此可知，镜像是由多层组织定义，这些层本质上就是文件。每层中具体的文件都存放在diff目录下。

继续看下镜像层目录中的内容
{{< highlight go "linenos=inline" >}}
tree 9266b15e0526e00dec39de79579edd308a6929851aa4cd6f1abdbee24086005b
9266b15e0526e00dec39de79579edd308a6929851aa4cd6f1abdbee24086005b
├── diff
│   └── var
│       └── cache
│           └── apt
│               ├── pkgcache.bin
│               └── srcpkgcache.bin
├── link
├── lower
└── work
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
cat 9266b15e0526e00dec39de79579edd308a6929851aa4cd6f1abdbee24086005b/link
XTSVQXPU33EB6U37TGSTZD6ERF

cat 9266b15e0526e00dec39de79579edd308a6929851aa4cd6f1abdbee24086005b/lower
l/5SCDQNUHYPUCLUEFYTFLJXRYIB
{{< /highlight >}}
可见，link文件的内容恰好是该层标志符的精简，对应l目录下的同名符号链接，而lower的内容描述了层序的组织关系，也就是说当前层的父层。

{{< highlight go "linenos=inline" >}}
mount | grep overlay
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
docker run -itd ubuntu
808e68fb68ba365eb176d5d85addf78f1d0be9c1650122b2cb9dc867a64a8127
{{< /highlight >}}

继续观察overlay2的挂载情况
{{< highlight go "linenos=inline" >}}
mount | grep overlay
overlay on /var/lib/docker/overlay2/9c5c19e4f0efed1bcc023ba42a909bb2e617a9f3dcc51ca98784125e8d77df77/merged type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/JFT2ROSB6TURDJ6HR4JDQNVPEW:/var/lib/docker/overlay2/l/N6GWFEGZOEYSV66LU2SWBHZ54T:/var/lib/docker/overlay2/l/MIA5PW7ES7CROKZ2IQKRRCZZ4M:/var/lib/docker/overlay2/l/XTSVQXPU33EB6U37TGSTZD6ERF:/var/lib/docker/overlay2/l/5SCDQNUHYPUCLUEFYTFLJXRYIB,upperdir=/var/lib/docker/overlay2/9c5c19e4f0efed1bcc023ba42a909bb2e617a9f3dcc51ca98784125e8d77df77/diff,workdir=/var/lib/docker/overlay2/9c5c19e4f0efed1bcc023ba42a909bb2e617a9f3dcc51ca98784125e8d77df77/work)
{{< /highlight >}}

从输出信息可知lowerdir，upperdir，workdir等信息，此时再观察下/var/lib/docker/overlay2目录
{{< highlight go "linenos=inline" >}}
ls /var/lib/docker/overlay2
06e334c99ed7523d23207331c69f33cae27ba27cba6aca792123a18294b7ff54  9c5c19e4f0efed1bcc023ba42a909bb2e617a9f3dcc51ca98784125e8d77df77       l
3ce856fe582ee3a07fc83315f11506baf4ec156dda2b3b2f2aef425d83578d2f  9c5c19e4f0efed1bcc023ba42a909bb2e617a9f3dcc51ca98784125e8d77df77-init
9266b15e0526e00dec39de79579edd308a6929851aa4cd6f1abdbee24086005b  a12e3822e217c5e8cb1f47533a9c845a15004ac239dd7e6fefd39d70ab6059ca
{{< /highlight >}}
这下多了两个目录，l子目录也是如此
{{< highlight go "linenos=inline" >}}
ll /var/lib/docker/overlay2/l
total 24
lrwxrwxrwx 1 root root 72 Nov 26 21:58 5SCDQNUHYPUCLUEFYTFLJXRYIB -> ../06e334c99ed7523d23207331c69f33cae27ba27cba6aca792123a18294b7ff54/diff
lrwxrwxrwx 1 root root 72 Nov 27 21:23 7T4R3QZKYDX7ABELKVP72CDVL5 -> ../9c5c19e4f0efed1bcc023ba42a909bb2e617a9f3dcc51ca98784125e8d77df77/diff
lrwxrwxrwx 1 root root 77 Nov 27 21:23 JFT2ROSB6TURDJ6HR4JDQNVPEW -> ../9c5c19e4f0efed1bcc023ba42a909bb2e617a9f3dcc51ca98784125e8d77df77-init/diff
lrwxrwxrwx 1 root root 72 Nov 26 21:59 MIA5PW7ES7CROKZ2IQKRRCZZ4M -> ../a12e3822e217c5e8cb1f47533a9c845a15004ac239dd7e6fefd39d70ab6059ca/diff
lrwxrwxrwx 1 root root 72 Nov 26 21:59 N6GWFEGZOEYSV66LU2SWBHZ54T -> ../3ce856fe582ee3a07fc83315f11506baf4ec156dda2b3b2f2aef425d83578d2f/diff
lrwxrwxrwx 1 root root 72 Nov 26 21:59 XTSVQXPU33EB6U37TGSTZD6ERF -> ../9266b15e0526e00dec39de79579edd308a6929851aa4cd6f1abdbee24086005b/diff
{{< /highlight >}}

9c5c19e4f0efed1bcc023ba42a909bb2e617a9f3dcc51ca98784125e8d77df77-init层是动态创建而成
{{< highlight go "linenos=inline" >}}
tree 9c5c19e4f0efed1bcc023ba42a909bb2e617a9f3dcc51ca98784125e8d77df77-init
9c5c19e4f0efed1bcc023ba42a909bb2e617a9f3dcc51ca98784125e8d77df77-init
├── diff
│   ├── dev
│   │   ├── console
│   │   ├── pts
│   │   └── shm
│   └── etc
│       ├── hostname
│       ├── hosts
│       ├── mtab -> /proc/mounts
│       └── resolv.conf
├── link
├── lower
└── work
    └── work
{{< /highlight >}}
可见，该层主要是一些基本的配置文件
{{< highlight go "linenos=inline" >}}
tree -L 2 9c5c19e4f0efed1bcc023ba42a909bb2e617a9f3dcc51ca98784125e8d77df77
9c5c19e4f0efed1bcc023ba42a909bb2e617a9f3dcc51ca98784125e8d77df77
├── diff
├── link
├── lower
├── merged
│   ├── bin
│   ├── boot
│   ├── dev
│   ├── etc
│   ├── home
│   ├── lib
│   ├── lib64
│   ├── media
│   ├── mnt
│   ├── opt
│   ├── proc
│   ├── root
│   ├── run
│   ├── sbin
│   ├── srv
│   ├── sys
│   ├── tmp
│   ├── usr
│   └── var
└── work
    └── work
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
cat 9c5c19e4f0efed1bcc023ba42a909bb2e617a9f3dcc51ca98784125e8d77df77/lower
l/JFT2ROSB6TURDJ6HR4JDQNVPEW:l/N6GWFEGZOEYSV66LU2SWBHZ54T:l/MIA5PW7ES7CROKZ2IQKRRCZZ4M:l/XTSVQXPU33EB6U37TGSTZD6ERF:l/5SCDQNUHYPUCLUEFYTFLJXRYIB
{{< /highlight >}}

另外，这个层多了一个merged目录


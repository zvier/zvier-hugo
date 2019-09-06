---
title: "containerd源码解析(1)——main.go"
date: 2019-08-31T21:27:13+08:00
draft: true
categories: ["docker"]
tags: ["containerd"]
---

# 概述
containerd的命令行入口文件main.go很简单，一个init函数，一个main函数。以下将分别展开解读  

# init
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/cmd/containerd/main.go
import (
    ...
    "github.com/containerd/containerd/cmd/containerd/command"
    "github.com/containerd/containerd/pkg/seed"
)

func init() {
    seed.WithTimeAndRand()
}
{{< /highlight >}}

# WithTimeAndRand
init函数中调用了pkg/seed包里的WithTimeAndRand函数
{{< highlight go "linenos=inline" >}}
github.com/containerd/containerd/pkg/seed/seed.go
// WithTimeAndRand seeds the global math rand generator with nanoseconds
// XOR'ed with a crypto component if available for uniqueness.
func WithTimeAndRand() {
    var (
        b [4]byte
        u int64
    )

    tryReadRandom(b[:])

    // Set higher 32 bits, bottom 32 will be set with nanos
    u |= (int64(b[0]) << 56) | (int64(b[1]) << 48) | (int64(b[2]) << 40) | (int64(b[3]) << 32)

    rand.Seed(u ^ time.Now().UnixNano())
}
{{< /highlight >}}
WithTimeAndRand函数的主要作用在于通过一个int64类型的值，初始化随机数生成器到一个确定的状态，不过正如其名，这个int64类型的seed值结合了时间和随机数，变量b有32bit，u有64bit，b通过调用tryReadRandom初始化后，通过移位操作，转变成u的高32位，然后u再与time.Now().UnixNano()异或生成自己的低32位。   

简单过一下tryReadRandom，它主要用于尝试从随机数源(比如/dev/urandom)中读取一个随机数，当然因为只是尝试，即便读取失败也没关系。  
{{< highlight go "linenos=inline" >}}
github.com/containerd/containerd/pkg/seed/seed_linux.go
import "golang.org/x/sys/unix"

func tryReadRandom(p []byte) {
    // Ignore errors, just decreases uniqueness of seed
    unix.Getrandom(p, unix.GRND_NONBLOCK)
}
{{< /highlight >}}

> /dev/random和/dev/urandom(u, unlocked，非阻塞的随机数发生器)文件都是linux系统中的随机伪设备，提供随机字节数据流(熵池)，这些数据流即当前系统的环境噪音，描述了一个系统的混乱程度，用命令: cat /dev/random | od -x 查看一下，可用看到该设备会源源不断的输出数据流。  

再来看一下Getrandom系统调用，它就是用来读取随机源中的数据流来填充目标缓冲的，声明如下:
{{< highlight go "linenos=inline" >}}
import "golang.org/x/sys/unix"
func Getrandom(buf []byte, flags int) (n int, err error)
{{< /highlight >}}
buf即待填充的目标缓冲区，flags参数可用控制Getrandom的行为，当设置为GRND_NONBLOCK时，Getrandom将以非阻塞的方式执行，如果random源中还没有可用的数据时，会立即返回。在目前笔者的系统中，该函数还没有实现，测试返回值为:(-1, function not implemented)，所以tryReadRandom调用之后，p依然为空。

> golang位运算   
1. << 左移  
2. >> 右移  
3. x ^ y 异或  
4. x | y 或  
5. x & y 与  
6. ^x 取反  

# main
现在进入main，简单来看，就是创建来一个app，然后根据命令行参数os.Args执行这个app，就这么简单  
{{< highlight go "linenos=inline" >}}
func main() {
    app := command.App()
    if err := app.Run(os.Args); err != nil {
        fmt.Fprintf(os.Stderr, "containerd: %s\n", err)
        os.Exit(1)
    }
}
{{< /highlight >}}


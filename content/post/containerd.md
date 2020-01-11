---
title: "containerd"
date: 2019-12-29T14:10:01+08:00
draft: true
categories: ["技术"]
tags: ["containerd"]
---
我居北海君南海，寄雁传书谢不能。桃李春风一杯酒，江湖夜雨十年灯。持家但有四立壁，治病不蕲三折肱。想见读书头已白，隔溪猿哭瘴溪藤。——黄庭坚《寄黄几复》
<!--more-->
# containerd架构
1.0之前，containerd的主要功能就是监控容器的运行状态，接收和处理容器事件，功能和结构都相对单一。

1.0之后，containerd被定位为容器管理的中间层，对外，containerd以Service的形态暴露GRPC接口，便于上游全面控制和管理容器生命周期。对内，containerd通过插件的方式提供容器存储和运行时管理相关的功能。

以下是containerd 1.0之后的新架构图:  
![contaienrd](/img/containerd/containerd_architecture.png)
可见，对外除了提供GRPC接口，还支持Metrics统计接口，用于统计容器相关的运行数据。  

新版containerd类似于一个微服务架构，内部拆分为Content，Snapshot，Diff，Images，Containers，Tasks，Events等Service，每个服务都以插件形式集成，其中Content，Snapshot，Diff和存储相关，Images，Containers和Metadata相关，Tasks和Events和Runtimes相关，除此之外还有一些其它辅助Service。

# 容器生命周期管理
## containerd
直观来说，容器生命周期的管理由containerd负责, containerd作为daemon运行在后台，对外提供容器管理API，但容器的生命周期又完全不依赖于containerd，如果containerd进程挂了，容器依然可以正常运行。  

在containerd中，一个容器就是一个Task，Task包含跑在容器中的进程Process。  
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/runtime/task.go
// Task is the runtime object for an executing container
type Task interface {
    Process

    // PID of the process
    PID() uint32
    // Namespace that the task exists in
    Namespace() string
    // Pause pauses the container process
    Pause(context.Context) error
    // Resume unpauses the container process
    Resume(context.Context) error
    // Exec adds a process into the container
    Exec(context.Context, string, ExecOpts) (Process, error)
    // Pids returns all pids
    Pids(context.Context) ([]ProcessInfo, error)
    // Checkpoint checkpoints a container to an image with live system data
    Checkpoint(context.Context, string, *types.Any) error
    // Update sets the provided resources to a running task
    Update(context.Context, *types.Any) error
    // Process returns a process within the task for the provided id
    Process(context.Context, string) (Process, error)
    // Stats returns runtime specific metrics for a task
    Stats(context.Context) (*types.Any, error)
}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/runtime/task.go
// Process is a runtime object for an executing process inside a container
type Process interface {
    // ID of the process
    ID() string
    // State returns the process state
    State(context.Context) (State, error)
    // Kill signals a container
    Kill(context.Context, uint32, bool) error
    // Pty resizes the processes pty/console
    ResizePty(context.Context, ConsoleSize) error
    // CloseStdin closes the processes stdin
    CloseIO(context.Context) error
    // Start the container's user defined process
    Start(context.Context) error
    // Wait for the process to exit
    Wait(context.Context) (*Exit, error)
    // Delete deletes the process
    Delete(context.Context) (*Exit, error)
}
{{< /highlight >}}

## containerd-shim
具体来说，容器的生命周期containerd将托付给containerd-shim来管理，启动后，每个容器都有一个叫做containerd-shim的父进程，shim进程将负责容器的启动, 回收容器进程, 监控容器OOM事件，上报容器退出事件, 管理容器的标准输入输出, 以及持有pty设备的master端。   

在containerd的PlatformRuntime定义了一组接口，用于实现对容器生命周期的控制，shim目前提供v1和v2两个版本，都实现了PlatformRuntime，containerd和containerd-shim之间通过GRPC接口通信，用户在运行时, 可以根据需要动态指定需要使用的shim版本。 
{{< highlight go "linenos=inline" >}}
ctr run --runtime io.containerd.runc.v1
ctr run --runtime io.containerd.runc.v2
{{< /highlight >}}

shim v1是containerd早期实现的一套shim接口，后期可能被废弃，而shim v2是为了让containerd兼容以虚拟机为容器而新增的一套shim接口规范。shim v2需要支持start和delete两个二进制命令接口, 并需要对应的命令行参数实现。另外，由于shim还需要关注容器的退出以及OOM事件，因此shim v2需要在容器退出时发布TaskExitEventTopic消息, 在容器出现OOM时需要发布TaskOOMEventTopic消息。

有了containerd-shim这么一层抽象管理容器生命周期，容器真正的运行时就可以是任何类型了，比如runc，lxc等容器运行时，也可以是kata, firecracker等虚拟化容器运行时。只要满足shim接口约束，便可以被containerd管理。 

# Content服务
Content服务用来管理镜像层和镜像内容，支持最新的Docker Registry HTTP API V2, Content的设计依照了Content-addressable storage的原则, 也就是说所有存储的内容都是以字节为单位进行摘要计算, 可以根据摘要值确定Content内容的完整性。Content模块实现了镜像完整性检测, 可恢复的镜像上传下载操作, 也就是可以支持断点续传。

# Bundle
对于shim来说, 它的主要输入叫做bundle, 主要包含配置文件和rootfs文件系统, 大致的目录结构如下
{{< highlight go "linenos=inline" >}}
├── io.containerd.runtime.vx
│   └── default
│       └── hello_world
│           ├── config.json
│           └── rootfs/
{{< /highlight >}}
其中default是命名空间。

# 容器创建数据流
Containerd会通过image服务, 拉取需要的镜像, 镜像分为元数据和真实内容两部分, 拉取的内容会存储到Content中;
镜像拉取完成后, Bundle控制器会根据元数据信息, 将Content的内容解压合并到Snapshot快照模块中进行挂载等操作;
Bundle模块控制Snapshot快照模块将Content的所有内容进行组合挂载, 将rootfs和配置信息准备好后, 将所有内容交给运行时模块;
运行时模块根据bundle内容启动并运行容器.
可以看到, 整个流程贯穿其中的是Events模块, Events模块负责收集各个模块运行信息.

# 参考
[Containerd 分析](https://heychenbin.github.io/post/containerd_intro/)  
[Containerd Content 服务](https://www.jianshu.com/p/24553c6977fd)


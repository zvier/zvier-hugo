---
title: "containerd shim详解 (二)"
date: 2019-06-23T16:07:40+08:00
draft: true
categories: ["docker"]
tags: [""]
---


# 命令行参数初始化——init

初始化部分主要是初始化containerd-shim命令行的flag，对照containerd-shim进程的启动命令，我们来看下这些flag的定义

{{< highlight shell >}}
containerd-shim -namespace moby \
-workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/0a5d429257468cd6465d11f70012dca244ce614ea15eb6f67b05a2a31a428425 \
-address /run/containerd/containerd.sock \
-containerd-binary /usr/bin/containerd \
-runtime-root /var/run/docker/runtime-runc
{{< /highlight >}}

* workdir:  bundle path，路径最后的id就是container id
* runtime-root:  runtime binary

{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/cmd/containerd-shim/main_unix.go
var (
    debugFlag            bool
    namespaceFlag        string
    socketFlag           string
    addressFlag          string
    workdirFlag          string
    runtimeRootFlag      string
    criuFlag             string
    systemdCgroupFlag    bool
    containerdBinaryFlag string

    bufPool = sync.Pool{
        New: func() interface{} {
            return bytes.NewBuffer(nil)
        },
    }
)

func init() {
    flag.BoolVar(&debugFlag, "debug", false, "enable debug output in logs")
    flag.StringVar(&namespaceFlag, "namespace", "", "namespace that owns the shim")
    flag.StringVar(&socketFlag, "socket", "", "abstract socket path to serve")
    flag.StringVar(&addressFlag, "address", "", "grpc address back to main containerd")
    flag.StringVar(&workdirFlag, "workdir", "", "path used to storge large temporary data")
    flag.StringVar(&runtimeRootFlag, "runtime-root", proc.RuncRoot, "root directory for the runtime")
    flag.StringVar(&criuFlag, "criu", "", "path to criu binary")
    flag.BoolVar(&systemdCgroupFlag, "systemd-cgroup", false, "set runtime to use systemd-cgroup")
    // currently, the `containerd publish` utility is embedded in the daemon binary.
    // The daemon invokes `containerd-shim -containerd-binary ...` with its own os.Executable() path.
    flag.StringVar(&containerdBinaryFlag, "containerd-binary", "containerd", "path to containerd binary (used for `containerd publish`)")
    flag.Parse()
}
{{< /highlight >}}

上面这段代码没什么特别的，值得一提的是sync.Pool的资源池的使用，可以参考[sync.Pool]()





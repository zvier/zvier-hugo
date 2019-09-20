---
title: "containerd-containerd-cmd-containerd-command-service_unsupported"
date: 2019-09-08T16:57:27+08:00
draft: true
categories: [""]
tags: ["containerd"]
---
# 简述
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/containerd/containerd/cmd/containerd/command/service_unsupported.go
{{< /highlight >}}

# 编译

关于选择性编译，有两个点需要注意:    

1. go build 会选择性地编译以系统名结尾的文件(linux、darwin、windows、freebsd)，例如在linux下，go build只会选择config_linux.go，其它系统命名后缀文件全部忽略    
2. 如果在xxx.go文件的文件头上添加<code>// +build !windows</code>，在windows系统上go build将选择不编译该文件  

{{< highlight go "linenos=inline" >}}
// +build !windows
{{< /highlight >}}

# serviceFlags
{{< highlight go "linenos=inline" >}}
// serviceFlags returns an array of flags for configuring containerd to run
// as a service. Only relevant on Windows.
func serviceFlags() []cli.Flag {
	return nil
}
{{< /highlight >}}

// applyPlatformFlags applys platform-specific flags.
func applyPlatformFlags(context *cli.Context) {
}

// registerUnregisterService is only relevant on Windows.
func registerUnregisterService(root string) (bool, error) {
	return false, nil
}

// launchService is only relevant on Windows.
func launchService(s *server.Server, done chan struct{}) error {
	return nil
}


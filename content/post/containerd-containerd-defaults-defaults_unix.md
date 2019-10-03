---
title: "Containerd Containerd Defaults Defaults_unix"
date: 2019-09-22T10:56:03+08:00
draft: true
categories: [""]
tags: [""]
---
# 简述
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/containerd/containerd/defaults/defaults_unix.go
{{< /highlight >}}

# 常量声明

{{< highlight go "linenos=inline" >}}
const (
	// DefaultRootDir is the default location used by containerd to store
	// persistent data
	DefaultRootDir = "/var/lib/containerd"
	// DefaultStateDir is the default location used by containerd to store
	// transient data
	DefaultStateDir = "/run/containerd"
	// DefaultAddress is the default unix socket address
	DefaultAddress = "/run/containerd/containerd.sock"
	// DefaultDebugAddress is the default unix socket address for pprof data
	DefaultDebugAddress = "/run/containerd/debug.sock"
	// DefaultFIFODir is the default location used by client-side cio library
	// to store FIFOs.
	DefaultFIFODir = "/run/containerd/fifo"
	// DefaultRuntime is the default linux runtime
	DefaultRuntime = "io.containerd.runc.v2"
)
{{< /highlight >}}

---
title: "containerd-containerd-platforms-defaults_unix"
date: 2019-09-22T11:05:03+08:00
draft: true
categories: ["技术"]
tags: ["containerd"]
---
# 简述
<!--more-->

# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/containerd/containerd/platforms/defaults_unix.go
{{< /highlight >}}

# Default
{{< highlight go "linenos=inline" >}}
// +build !windows
// Default returns the default matcher for the platform.
func Default() MatchComparer {
	return Only(DefaultSpec())
}
{{< highlight go "linenos=inline" >}}


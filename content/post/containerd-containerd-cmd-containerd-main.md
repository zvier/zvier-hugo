---
title: "containerd-containerd-cmd-containerd-main"
date: 2019-09-08T16:12:09+08:00
draft: true
categories: [""]
tags: ["containerd"]
---
# 简述
containerd命令入口
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/containerd/containerd/cmd/containerd/main.go
{{< /highlight >}}

# init
随机数种子初始化
{{< highlight go "linenos=inline" >}}
import (
	"github.com/containerd/containerd/cmd/containerd/command"
	"github.com/containerd/containerd/pkg/seed"
)
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
func init() {
	seed.WithTimeAndRand()
}
{{< /highlight >}}
## 引用说明
1. 随机数种子初始化: [seed.WithTimeAndRand](http://www.zvier.top/post/containerd-containerd-pkg-seed-seed/#withtimeandrand)

# main
命令行入口，创建一个App，并执行
{{< highlight go "linenos=inline" >}}
func main() {
	app := command.App()
	if err := app.Run(os.Args); err != nil {
		fmt.Fprintf(os.Stderr, "containerd: %s\n", err)
		os.Exit(1)
	}
}
{{< /highlight >}}

## 引用说明
1. 创建App: [command.App](http://www.zvier.top/post/containerd-containerd-cmd/-containerd-command-main/#app)

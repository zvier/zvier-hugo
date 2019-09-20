---
title: "Calico Projectcalico Cni Plugin Cmd Calico Calico"
date: 2019-09-08T10:47:11+08:00
draft: true
categories: [""]
tags: ["calico"]
---
# 简述
cni插件calico命令行入口
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/projectcalico/cni-plugin/cmd/calico/calico.go
{{< /highlight >}}

# main
main为calico命令的入口函数，根据不同的版本启动对应的插件逻辑，其中VERSION变量的值是在版本编译时通过<code>-ldflags "-X main.VERSION=$(GIT_VERSION)</code>注入的
{{< highlight go "linenos=inline" >}}
import (
	"github.com/projectcalico/cni-plugin/pkg/plugin"
)
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// VERSION is filled out during the build process (using git describe output)
var VERSION string

func main() {
	plugin.Main(VERSION)
}
{{< /highlight >}}

## 引用说明
1. 插件启动: [plugin.Main](http://www.zvier.top/post/calico-projectcalico-cni-plugin-pkg-plugin-plugin/#main)

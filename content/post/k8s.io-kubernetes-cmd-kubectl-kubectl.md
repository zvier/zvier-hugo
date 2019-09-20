---
title: "k8s.io-kubernetes-cmd-kubectl-kubectl"
date: 2019-09-07T07:50:10+08:00
draft: true
categories: [""]
tags: ["k8s"]
---
# 简述
kubectl命令行操作的入口

# 文件
{{< highlight go "linenos=inline" >}}
k8s.io/kubernetes/cmd/kubectl/kubectl.go
{{< /highlight >}}

# main
从kubectl main函数的视角来看，逻辑还是很简单的，分为以下几个步骤:  
1. 随机数种子初始化  
2. 生成命令行对象command  
3. 安装命令行的flag  
4. 初始化日志系统  
5. 然后执行命令行  
{{< highlight go "linenos=inline" >}}
cliflag "k8s.io/component-base/cli/flag"
"k8s.io/kubernetes/pkg/kubectl/cmd"
"k8s.io/kubernetes/pkg/kubectl/util/logs"
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// k8s.io/kubernetes/cmd/kubectl/kubectl.go
func main() {
    rand.Seed(time.Now().UnixNano())

    command := cmd.NewDefaultKubectlCommand()

    // TODO: once we switch everything over to Cobra commands, we can go back to calling
    // cliflag.InitFlags() (by removing its pflag.Parse() call). For now, we have to set the
    // normalize func and add the go flag set by hand.
    pflag.CommandLine.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
    pflag.CommandLine.AddGoFlagSet(goflag.CommandLine)
    // cliflag.InitFlags()
    logs.InitLogs()
    defer logs.FlushLogs()

    if err := command.Execute(); err != nil {
        fmt.Fprintf(os.Stderr, "%v\n", err)
        os.Exit(1)
    }
}
{{< /highlight >}}
# 引用说明
1. 创建kubectl命令行: [cmd.NewDefaultKubectlCommand](http://www.zvier.top/post/k8s.io-kubernetes-pkg-kubectl-cmd-cmd/#newdefaultkubectlcommand)

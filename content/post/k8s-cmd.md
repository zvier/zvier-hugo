---
title: "k8s 命令行"
date: 2020-01-20T17:27:58+08:00
draft: true
categories: ["技术"]
tags: ["k8s"]
---
<!--more-->
# get命令
get命令用于获取集群的一个或一些resource信息，resource包括集群节点、运行的pod，ReplicationController，service等。  

1. 查看所有的pods
{{< highlight go >}}
kubectl get po -A
kubectl get po --all-namespaces
kubectl get po --all-namespaces -o wide
{{< /highlight >}}

1. 查看所有的节点
{{< highlight go >}}
kubectl get nodes
{{< /highlight >}}

1. 查看资源详情
{{< highlight go >}}
kubectl get nodes hd1 -o yaml
kubectl get nodes hd1 -o json
kubectl get pods kube-flannel-ds-amd64-9wpv2 -n kube-system -o yaml
kubectl get pods kube-flannel-ds-amd64-9wpv2 -n kube-system -o json
{{< /highlight >}}

# describe命令
describe类似于get，同样用于获取resource的相关信息，不同的是，get获得的是更详细的resource个性的详细信息，describe获得的是resource集群相关的信息   
{{< highlight go >}}
kubectl describe nodes hd1
kubectl describe pods kube-flannel-ds-amd64-9wpv2 -n kube-system
{{< /highlight >}}

# create命令
{{< highlight go >}}
kubectl create -f kube-flannel.yaml
{{< /highlight >}}

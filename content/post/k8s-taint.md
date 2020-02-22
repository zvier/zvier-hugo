---
title: "k8s taint and toleration"
date: 2020-01-13T15:50:00+08:00
draft: true
categories: ["技术"]
tags: ["k8s"]
---
来如流水兮逝如风，不知何处来兮何所终
<!--more-->
# 简述
k8s中，可以通过定义taint(污点)或toleration(容忍)达到pod调度对node排斥或容忍的目的。比如kubeadm初始化集群时，出于安全考虑Pod就不会被调度到Master节点上，换句话说，就是此时Master节点不参与工作负载，这正是通过给Master节点打上了key为node-role.kubernetes.io，value为master，effect为NoSchedule的污点实现。

# 查看master节点的污点
k8s集群中的master节点，默认是不会有任务调度到的
{{< highlight go "linenos=inline" >}}
kubectl get nodes
NAME   STATUS     ROLES    AGE   VERSION
hd1    Ready      <none>   21h   v1.17.0
hd2    Ready      master   23h   v1.17.0
hd3    Ready      <none>   21h   v1.17.0
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
kubectl describe nodes hd2 | grep -E '(Roles|Taints)'
Roles:              master
Taints:             node-role.kubernetes.io/master:NoSchedule
{{< /highlight >}}
可见master节点hd2拥有key为: node-role.kubernetes.io，值为: master，effect为: NoSchedule的taint

## 添加污点
taint命令格式
{{< highlight go "linenos=inline" >}}
kubectl taint node $node $key=$value:$effect
kubectl taint node $node $key=:$effect       // 此时value值为空，但冒号不能省略
{{< /highlight >}}


其中，taint的effect有以下三种:  

* NoSchedule: 一定不能被调度到，仅影响新来的调度，对现有的Pod不产生影响  
* NoExecute: 不仅不会调度到，还也影响现有的Pod，现有不能容忍的Pod将被驱逐到其它节点  
* PreferNoSchedule: 尽量不要调度到，NoSchedule的柔性版本，实在没地方运行的情况下也能调过来  

给节点添加key为$key，value为$value的taint，只有能容忍这个taint的Pod才能分配到该节点上  

{{< highlight go "linenos=inline" >}}
kubectl taint node hd1 $key=$value:NoExecute
kubectl taint node hd2 $key=$value:NoExecute
kubectl taint node hd3 $key=$value:NoExecute
{{< /highlight >}}

为node添加taint后，如果该节点上的Pod不能忍受effect值为NoExecute的taint，那么这些Pod将马上被驱逐，如果Pod能够忍受effect值为NoExecute的taint，但是在toleration定义中没有指定tolerationSeconds，则Pod还会一直在这个节点上运行，如果Pod能够忍受effect值为NoExecute的taint，而且指定了tolerationSeconds，则Pod还能在这个节点上继续运行这个指定的时间


给master节点设置taint
{{< highlight go "linenos=inline" >}}
kubectl taint node hd2 node-role.kubernetes.io/master=:NoSchedule
{{< /highlight >}}
为master设置的这个taint中, node-role.kubernetes.io/master为key, value为空, effect为NoSchedule



## 删除污点
执行
{{< highlight go "linenos=inline" >}}
kubectl taint node hd1 $key:NoSchedule-  // 删除$key下指定的effect，这里可以不用指定value
kubectl taint node hd1 $key:NoExecute-   
kubectl taint node hd1 $key-             // 删除$key下所有的effect
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
kubectl taint node --all node-role.kubernetes.io/master-
{{< /highlight >}}
返回  
{{< highlight go "linenos=inline" >}}
node/<your-hostname> untainted
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
kubectl describe nodes hd1 | grep -E '(Roles|Taints)'
Roles:              <none>
Taints:             <none>
{{< /highlight >}}

可以通过在Pod的yaml文件中定义toleration字段来容忍节点的污点，Pod的调度将匹配key，value所指定的taint

# 添加容忍
如果master节点设置了key为node-role.kubernetes.io/master=:NoSchedule的taint, 则需要为Pod的yaml文件中添加如下spec.tolerations配置, 才能让该Pod容忍master节点的这个污点
{{< highlight go "linenos=inline" >}}
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx-pod
spec:
  containers:
  - name: nginx-assistant
    image: nginx
    resources:
      limits:
        cpu: 30m
        memory: 20Mi
      requests:
        cpu: 20m
        memory: 10Mi
  tolerations:
  - key: "node-role.kubernetes.io/master"
    value: ""
    operator: Equal
    effect: "NoSchedule"
{{< /highlight >}}
kubectl apply后，可以看到master节点也可以接受Pod的调度，而未被定义容忍度的Pod将被驱逐到其它节点，如果没有匹配到对应的污点，则会调度到未配置污点的节点上

# 基于taint的驱逐
当节点出现问题时，我们希望能驱逐该节点上的Pod，这时可以基于taint来完成对Pod的驱逐。当某种条件为真时，NodeController会自动给节点添加一个taint，当Pod不能容忍该taint时，就会被驱逐。

当前内置的taint有如下：

* node.kubernetes.io/not-ready：节点未准备好  
* node.kubernetes.io/out-of-disk：节点磁盘耗尽  
* node.kubernetes.io/memory-pressure：节点存在内存压力  
* node.kubernetes.io/disk-pressure：节点存在磁盘压力  
* node.kubernetes.io/network-unavailable：节点网络不可用  

当节点出现问题时，NodeController会自动给节点添加相应的taint，从而完成对Pod的驱逐

为了保证由于节点问题引起的Pod驱逐rate limiting行为正常，系统实际上会以rate-limited的方式添加taint。在像master和node通讯中断等场景下，这避免了Pod被大量驱逐。使用这个特性，结合tolerationSeconds，Pod就可以指定当节点出现一个或全部上述问题时，还将在这个节点上运行多长的时间。

除了我们定义的容忍匹配的 taint 外，还默认匹配了 not-ready，unreachable 这两个 taint，并且指定 tolerationSeconds 为 5 分钟。这种自动添加 toleration 机制保证了在其中一种问题被检测到时 pod 默认能够继续停留在当前节点运行 5 分钟；这两个默认 toleration 是由 DefaultTolerationSeconds admission controller添加的。

另外：我们可以指定这个时间，在网络断开时，仍然希望停留在当前节点上运行一段较长的时间，愿意等待网络恢复以避免被驱逐。在这种情况下，pod 的 toleration 可能是下面这样的：
tolerations:
- key: "node.alpha.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000

DaemonSet中的Pod被创建时，针对taint自动添加的NoExecute的toleration将不会指定tolerationSeconds。

比如：系统pod（canal，dns等）不会指定 tolerationSeconds

kubectl describe pod/canal-75pct -n kube-system

Tolerations:     :NoSchedule
                 :NoExecute
                 CriticalAddonsOnly
                 node.kubernetes.io/disk-pressure:NoSchedule
                 node.kubernetes.io/memory-pressure:NoSchedule
                 node.kubernetes.io/network-unavailable:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute
                 node.kubernetes.io/unreachable:NoExecute
                 node.kubernetes.io/unschedulable:NoSchedule
这保证了当节点出现上述问题时，DaemonSet中的Pod永远不会被驱逐，这和TaintBasedEvictions 这个特性被禁用后的行为是一样的。

# 参考
[Kubernetes 配置 Taint 和 Toleration(污点和容忍)](https://www.cnblogs.com/weavepub/p/10968667.html)
[kubernetes的taint（污点）和toleration（容忍）](https://blog.csdn.net/fanren224/article/details/86610741)
[为k8s-master节点添加污点taints](https://docs.lvrui.io/2018/11/14/%E4%B8%BAk8s-master%E8%8A%82%E7%82%B9%E6%B7%BB%E5%8A%A0%E6%B1%A1%E7%82%B9taints/)

---
title: "k8s 部署"
date: 2020-01-12T10:11:26+08:00
draft: true
categories: ["技术"]
tags: ["k8s"]
---
<!--more-->
# 修改主机名(可选)
{{< highlight go "linenos=inline" >}}
hostnamectl set-hostname hd1
hostnamectl set-hostname hd2
hostnamectl set-hostname hd3
{{< /highlight >}}

# 主机配置
## 主机路由
{{< highlight go >}}
cat >> /etc/hosts <<EOF
172.16.123.194 hd1
172.16.124.229 hd2
172.16.116.163 hd3
EOF
{{< /highlight >}}

## 安全配置
1. 简单起见，关闭所有节点的防火墙
{{< highlight go >}}
systemctl stop firewalld  
systemctl disable firewalld
{{< /highlight >}}
2. 禁用selinux
临时禁用
{{< highlight go >}}
setenforce 0
{{< /highlight >}}
永久禁用
{{< highlight go >}}
vim /etc/selinux/config
SELINUX=disabled
{{< /highlight >}}

## 关闭交换分区
使用如下命令可确认swap是否关闭
{{< highlight go >}}
free -m
或
free -h
{{< /highlight >}}
如果swap没关闭，执行如下命令
{{< highlight go >}}
swapoff -a
{{< /highlight >}}
关闭swap也可以通过修改kubelet启动参数仅作用kubelet
{{< highlight go >}}
cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
{{< /highlight >}}
可以看到kubelet的配置文件为: --config=/var/lib/kubelet/config.yaml，该文件为kubeadm初始化集群时自动生成。所以只能通过修改/etc/sysconfig/kubelet文件的KUBELET_EXTRA_ARGS
{{< highlight go >}}
KUBELET_EXTRA_ARGS=--fail-swap-on=false
{{< /highlight >}}

# 配置iptables参数
在CentOS7上，由于iptables被绕过可能导致流量路由不正确，需要创建/etc/sysctl.d/k8s.conf文件，添加如下内容：
{{< highlight go >}}
cat > /etc/sysctl.d/k8s.conf <<EOF
vm.swappiness = 0
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
{{< /highlight >}}
生效/etc/sysctl.d/k8s.conf配置
{{< highlight go >}}
sysctl -p /etc/sysctl.d/k8s.conf
{{< /highlight >}}
加载并确认br_netfilter模块，启用此内核模块，以便遍历桥的数据包由iptables进行处理以进行过滤和端口转发，
{{< highlight go >}}
modprobe br_netfilter
lsmod | grep br_netfilter
{{< /highlight >}}

# 加载ipvs相关模块
ipvs已经加入到内核主干，所以为kube-proxy开启ipvs的前提加载以下的内核模块，可以在所有k8s节点执行以下脚本，保证节点重启后自动加载所有模块。
{{< highlight go >}}
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#! /bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
{{< /highlight >}}
执行脚本，并使用lsmode命令查看是否已经正确加载所需的内核模块。
{{< highlight go >}}
chmod 755 /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
{{< /highlight >}}

# 安装ipset和ipvsadm工具(可选)
{{< highlight go >}}
yum install ipset ipvsadm -y
{{< /highlight >}}

# 安装docker
k8s 1.6开始使用CRI容器运行时接口，内置在kubelet的dockershim中，默认容器运行时为docker
{{< highlight go >}}
wget http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum list docker-ce --showduplicates | sort -r
yum install -y docker-ce-19.03.5-3.el7
systemctl start docker  
systemctl enable docker
{{< /highlight >}}

确认一下iptables filter表中FOWARD链的默认策略(pllicy)为ACCEPT
{{< highlight go >}}
iptables -nvL

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
{{< /highlight >}}
如果不是ACCEPT，将会影响到k8s集群中Node和Pod的通信，需要执行以下命令改变
{{< highlight go >}}
iptables -P FORWARD ACCEPT
{{< /highlight >}}

# 安装k8s
## 配置k8s yum源
{{< highlight go >}}
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
{{< /highlight >}}
## 安装
{{< highlight go >}}
yum list kubeadm --showduplicates | sort -r
yum list kubelet --showduplicates | sort -r
yum list kubectl --showduplicates | sort -r
yum install -y kubeadm-1.17.0-0
yum install -y kubelet-1.17.0-0
yum install -y kubectl-1.17.0-0
systemctl enable kubelet && systemctl start kubelet
{{< /highlight >}}

# 跨VPC部署
如果是跨VPC部署，由于node在加入集群时，被登记的网卡ip很可能是内网ip，这时master节点和node节点通信就存在问题，再个人部署时，可以使用iptables做NAT转发
{{< highlight go >}}
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
iptables -t nat -A OUTPUT -p all -d ${internal-ip} -j DNAT --to-destination ${public-ip}
{{< /highlight >}}

# 部署master节点
## master节点初始化
这里执行初始化用到了--image-repository选项，指定初始化时从阿里云镜像仓库拉取所需要的镜像
{{< highlight go >}}
kubeadm init \
    --apiserver-advertise-address=${public-ip} \
    --image-repository registry.aliyuncs.com/google_containers \
    --kubernetes-version v1.17.3 \
    --pod-network-cidr=10.244.0.0/16 \
    // --ignore-preflight-errors=NumCPU,Swap
{{< /highlight >}}

## 参数说明
{{< highlight go >}}
--apiserver-advertise-address  
{{< /highlight >}}  

> 指明使用用master的哪个interface作为apiserver服务的地址。如果master有多个interface，建议明确指定，如果不指定，kubeadm会自动选择有默认网关的interface。  

{{< highlight go >}}
--pod-network-cidr
{{< /highlight >}}  

> 指定pod的网络范围。kubernetes支持多种网络方案，不同网络方案对--pod-network-cidr也有自己的要求，如果选择flannel作为pod网络插件，这里设置为10.244.0.0/16。  


{{< highlight go >}}
--image-repository
{{< /highlight >}}  

> kubenetes默认的registries地址是k8s.gcr.io，在国内访问不了，1.13版本后可以通过增加-–image-repository参数自定义镜像仓库，这里将其指定为阿里云镜像地址: registry.aliyuncs.com/google_containers。  

{{< highlight go >}}
--kubernetes-version=v1.17.0  
{{< /highlight >}}

> 关闭版本探测，因为它的默认值是stable-1，会导致从https://dl.k8s.io/release/stable-1.txt下载最新的版本号，我们可以将其指定给定版本跳过网络版本探测。

命令执行完后注意记录下初始化结果中的kubeadm join命令，部署worker节点时会用到

初始化过程说明如下：

1. [preflight] kubeadm 执行初始化前的检查。这里可以用--ignore-preflight-errors参数跳过部分错误
2. [kubelet-start] 生成kubelet的配置文件”/var/lib/kubelet/config.yaml”
3. [certificates] 生成相关的各种token和证书
4. [kubeconfig] 生成 KubeConfig 文件，kubelet 需要这个文件与 Master 通信
5. [control-plane] 安装 Master 组件，会从指定的 Registry 下载组件的 Docker 镜像。
6. [bootstraptoken] 生成token记录下来，后边使用kubeadm join往集群中添加节点时会用到
7. [addons] 安装附加组件 kube-proxy 和 kube-dns。
8. kubernetes master初始化成功，提示如何配置常规用户使用kubectl访问集群。
9. 提示如何安装pod网络。
10. 提示如何注册其他节点到cluster。

以上步骤也可以通过配置方式完成:
{{< highlight bash >}}
kubeadm init --config kubeadm.yaml
{{< /highlight >}}
其中kubeadm.yaml文件的生产可以通过如下命令生产:
{{< highlight bash >}}
kubeadm config print init-defaults
{{< /highlight >}}

然后根据实际情况调整部分参数
{{< highlight bash >}}
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
controlPlaneEndpoint: ${public-ip}:6443
clusterName: kubernetes
apiServer:
  certSANS:
  - ${public-ip}
  extraArgs:
    authorization-mode: Node,RBAC
    advertise-address: ${public-ip}
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
{{< /highlight >}}

# master配置
master初始化完后会提示如下信息:
{{< highlight go >}}
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
{{< /highlight >}}

{{< highlight go >}}
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
{{< /highlight >}}
上面这份config配置是为kubectl命令准备，config中保存着server地址，密钥等信息，按照提示操作将config文件保存到任何client机的任何账户下，便可以执行kubectl命令。所有操作是在client机上完成，这里的client机也可以由master或node节点充当。

{{< highlight go >}}
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/
{{< /highlight >}}

{{< highlight go >}}
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join ${ip}:6443 --token u2pv3g.sodo7rgsfoqc7s5u \
    --discovery-token-ca-cert-hash sha256:d65fc098e3ae98bc02ee670d0c8bfb266bbfe24d6433604b00f189f4797ae96b
{{< /highlight >}}
如果token忘记，可以执行如下命令获取
{{< highlight go >}}
kubeadm token create –print-join-command
{{< /highlight >}}
完后查看集群状态，确认各个组件都处于health状态
{{< highlight go >}}
kubectl get cs
{{< /highlight >}}

# 移除节点
master节点上执行
{{< highlight go >}}
kubectl drain hd1 --delete-local-data --force --ignore-daemonsets
kubectl delete node hd1
{{< /highlight >}}

节点上执行reset操作后，记得清理残留网卡信息
{{< highlight go >}}
kubeadm reset
{{< /highlight >}}

{{< highlight go >}}
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1

rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /etc/cni/
{{< /highlight >}}

# cni网络部署
根据master节点初始化信息提示，部署集群pod网络
{{< highlight go >}}
kubectl apply -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml
{{< /highlight >}}
或
{{< highlight go >}}
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml
{{< /highlight >}}
或直接
{{< highlight go >}}
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
{{< /highlight >}}
观察部署进度，直到相关pod状态均为Running
{{< highlight go >}}
watch kubectl get pods --all-namespaces
{{< /highlight >}}
如果集群节点为多网卡节点，可以修改kube-flannel.yml文件，通过--iface参数指定节点网卡，否则可能出现dns无法解析
{{< highlight go >}}
......
containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.10.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=eth1
......
{{< /highlight >}}
部署完后，可以通过如下命令查看集群中的daemonset
{{< highlight go >}}
kubectl get ds -l app=flannel -n kube-system
NAME                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE    CONTAINERS     IMAGES                                   SELECTOR
kube-flannel-ds-amd64     2         2         1       2            1           <none>          7d9h   kube-flannel   quay.io/coreos/flannel:v0.11.0-amd64     app=flannel
kube-flannel-ds-arm       0         0         0       0            0           <none>          7d9h   kube-flannel   quay.io/coreos/flannel:v0.11.0-arm       app=flannel
kube-flannel-ds-arm64     0         0         0       0            0           <none>          7d9h   kube-flannel   quay.io/coreos/flannel:v0.11.0-arm64     app=flannel
kube-flannel-ds-ppc64le   0         0         0       0            0           <none>          7d9h   kube-flannel   quay.io/coreos/flannel:v0.11.0-ppc64le   app=flannel
kube-flannel-ds-s390x     0         0         0       0            0           <none>          7d9h   kube-flannel   quay.io/coreos/flannel:v0.11.0-s390x     app=flannel
{{< /highlight >}}
可见，fannel的部署yaml文件是要在集群中创建5个针对不同平台的DaemonSet，但DESIRED值会根据节点的不同而不同。再来看下ds的yaml描述:
{{< highlight go >}}
kubectl get ds kube-flannel-ds-amd64 -n kube-system -o yaml
{{< /highlight >}}
{{< highlight go >}}
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: flannel
        tier: node
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: beta.kubernetes.io/os
                operator: In
                values:
                - linux
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - amd64
      ......
      tolerations:
      - effect: NoSchedule
        operator: Exists
{{< /highlight >}}
查看相关pod状态，确保所有pod处于Running状态
{{< highlight go >}}
kubectl get pod --all-namespaces -o wide
{{< /highlight >}}
注意，如果是公有云主机部署，flannel默认走内网ip，需要修正
{{< highlight go >}}
kubectl annotate node hd1 --overwrite flannel.alpha.coreos.com/public-ip=${public_ip}
{{< /highlight >}}

# taint master
kubeadm初始化的集群，出于安全考虑，pod不会调度到master节点上，即master节点不参与工作负载，这是因为master节点被打上了node-role.kubernetes.io/master:NoSchedule的污点
{{< highlight go >}}
kubectl describe node hd2 | grep Taint
{{< /highlight >}}
可以去掉这个污点，使得master节点参与工作负载
{{< highlight go >}}
kubectl taint nodes --all node-role.kubernetes.io/master-
{{< /highlight >}}
确认集群中各个节点的状态
{{< highlight go >}}
kubectl get nodes -o wide
{{< /highlight >}}

# 测试DNS
{{< highlight go >}}
kubectl run busybox -it --rm --image=radial/busybox sh
{{< /highlight >}}
进入后执行nslookup kubernetes.default确认解析正常:
{{< highlight go >}}
nslookup kubernetes.default

Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
{{< /highlight >}}

kubectl annotate node hd1 flannel.alpha.coreos.com/public-ip-overwrite=47.97.180.60
kubectl annotate node hd2 flannel.alpha.coreos.com/public-ip-overwrite=47.111.157.189
kubectl annotate node hd3 flannel.alpha.coreos.com/public-ip-overwrite=47.98.133.32

# 参考
[使用kubeadm安装Kubernetes 1.12](https://blog.frognew.com/2018/10/kubeadm-install-kubernetes-1.12.html)

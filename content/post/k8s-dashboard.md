---
title: "k8s dashboard"
date: 2019-10-27T18:38:36+08:00
draft: true
categories: ["技术"]
tags: ["k8s"]
---
浪花有意千重雪， 桃李无言一队春。 一壶酒，一竿纶， 世上似侬有几人？——李煜
<!--more-->
#  kubernetes dashboard 部署
获取kubernetes-dashboard.yaml文件
{{< highlight bash "linenos=inline" >}}
wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
kubectl create -f kubernetes-dashboard.yaml
{{< /highlight >}}
当然也可以
{{< highlight bash "linenos=inline" >}}
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
{{< /highlight >}}

# kubernetes-dashboard.yaml解读
kubernetes-dashboard.yaml文件中定义了Secret，Service Account，Role，Role Binding，Deployment，Service等资源
## Dashboard Secret
Dashboard的密钥，类型是Opaque(adj. 不透明的)，Opaque使用base64编码存储信息，可以通过base64 --decode解码就可以获得原始数据，安全性较弱。
{{< highlight bash "linenos=inline" >}}
# ------------------- Dashboard Secret ------------------- #

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque

---
{{< /highlight >}}
## Dashboard Service Account
该部分定义了Dashboard的用户，类型为Service Account，名称为kubernetes-dashboard
{{< highlight bash "linenos=inline" >}}
# ------------------- Dashboard Service Account ------------------- #

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system

---
{{< /highlight >}}

# Dashboard Role & Role Binding
该部分定义了Dashboard将使用的角色kubernetes-dashboard-minimal，rules中列出了角色所拥有的多个权限。总体来看，级别还是比较低的。接着又定义了一个Role Binding，将roleRef指定的角色kubernetes-dashboard-minimal与subjects指定的主体kubernetes-dashboard绑定。
{{< highlight bash "linenos=inline" >}}
# ------------------- Dashboard Role & Role Binding ------------------- #

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
rules:
  # Allow Dashboard to create 'kubernetes-dashboard-key-holder' secret.
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create"]
  # Allow Dashboard to create 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs"]
  verbs: ["get", "update", "delete"]
  # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["kubernetes-dashboard-settings"]
  verbs: ["get", "update"]
  # Allow Dashboard to get metrics from heapster.
- apiGroups: [""]
  resources: ["services"]
  resourceNames: ["heapster"]
  verbs: ["proxy"]
- apiGroups: [""]
  resources: ["services/proxy"]
  resourceNames: ["heapster", "http:heapster:", "https:heapster:"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard-minimal
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system

---
{{< /highlight >}}
## Dashboard Deployment
{{< highlight bash "linenos=inline" >}}
# ------------------- Dashboard Deployment ------------------- #

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
      - name: kubernetes-dashboard
        image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
          # Uncomment the following line to manually specify Kubernetes API server Host
          # If not specified, Dashboard will attempt to auto discover the API server and connect
          # to it. Uncomment only if the default does not work.
          # - --apiserver-host=http://my-address:port
        volumeMounts:
        - name: kubernetes-dashboard-certs
          mountPath: /certs
          # Create on-disk volume to store exec logs
        - mountPath: /tmp
          name: tmp-volume
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: kubernetes-dashboard-certs
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule

---
{{< /highlight >}}
可以看到，Dashboard的Deployment指定了其使用的ServiceAccount是kubernetes-dashboard。并且还将Secret kubernetes-dashboard-certs通过volumes挂在到pod内部的/certs路径。为何要挂载Secret ？原因是创建Secret 时会自动生成token。请注意参数--auto-generate-certificates，其表示Dashboard会自动生成证书。

## Dashboard Service
{{< highlight bash "linenos=inline" >}}
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard`
{{< /highlight >}}
Dashboard的Service定义了服务的类型以及端口等，可以修改以下，使用NodePort类型做映射，使得能使用主机端口访问dashboard。
{{< highlight bash "linenos=inline" >}}
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard`
{{< /highlight >}}
重新apply kubernetes-dashboard.yaml后，可以查看一下kubernetes-dashboard容器是否已经运行起来
{{< highlight bash "linenos=inline" >}}
kubectl get pods -n kube-system | grep 'kubernetes-dashboard'
{{< /highlight >}}


# 相关证书介绍
1. ca-key.pem 根私钥
2. ca.pem 根证书
3. kubernetes-key.pem 集群私钥
4. kubernetes.pem 集群证书
5. kube-proxy.pem proxy私钥-node节点进行认证
6. kube-proxy-key.pem proxy证书-node节点进行认证
7. admin.pem 管理员私钥-主要用于kubectl认证
8. admin-key.pem 管理员证书-主要用于kubectl认证
## TLS
TLS 的作用就是对通讯加密，防止中间人窃听；同时如果证书不信任的话根本就无法与 apiserver 建立连接，更不用提有没有权限向 apiserver 请求指定内容。

# 授予dashboard账户集群管理权限
使用kubeadm搭建k8s集群会默认开启RABC(角色访问控制机制)，k8s中的RBAC规定了一个用户或者用户组具有请求哪些api的权限。以下先了解几个基本概念:

## 用户
关于k8s的用户有两种: User和Service Account，User分配给管理员，Service Account分配给进程，这里的Dashboard就是一个进程，可以为其创建一个Service Account。

## 角色
角色Role是一系列权限的集合，例如一个Role可包含读取和列出Pod的权限，ClusterRole和Role 类似，其权限范围是整个集群。

## 角色绑定
RoleBinding把角色关联或映射到用户，即将角色和用户绑定，让用户拥有该角色设定的权限，同理ClusterRoleBinding和RoleBinding类似，可让用户拥有ClusterRole的权限。

## Secret
Secret是一个包含少量敏感信息如密码，令牌，或秘钥的对象。把这些信息保存在 Secret对象中，可以在这些信息被使用时加以控制，并可以降低信息泄露的风险。

在配合TLS加密使用的时候，客户端如果想要与apiserver通讯，就必须采用由apiserver CA签发的证书，形成信任关系，建立TLS连接。apiserver通过读取客户端证书的CN字段作为RBAC的用户名，读取O字段作为RBAC的用户组，进而访问服务。

以下通过编写配置文件kubernetes-dashboard-admin-rbac.yaml，分别创建账户以及为账户分配权限

{{< highlight go "linenos=inline" >}}
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kube-system
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
kubectl create -f kubernetes-dashboard-admin.rbac.yaml
{{< /highlight >}}

以上步骤简单等效于:
* 创建账户
{{< highlight go "linenos=inline" >}}
kubectl create serviceaccount kubernetes-dashboard-admin -n kube-system
{{< /highlight >}}
* 绑定权限
{{< highlight go "linenos=inline" >}}
kubectl create clusterrolebinding kubernetes-dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
{{< /highlight >}}

# 登陆
## 使用管理员角色登陆
* 获取账户kubernetes-dashboard-admin的secret
{{< highlight go "linenos=inline" >}}
kubectl -n kube-system get secret | grep kubernetes-dashboard-admin
{{< /highlight >}}
* 获取secret中的token信息
{{< highlight go "linenos=inline" >}}
kubectl describe secrets $secret -n kube-system | grep token
{{< /highlight >}}

Dashboard访问方式有4种: NodePort，API Server，kubectl proxy，Ingress
## 使用NodePort
使用NodePort时，需要配置好一个私有证书或者使用共有证书，否则会提示证书错误NET::ERR_CERT_INVALID
1. 查看kubernetes-dashboard 容器所在节点
{{< highlight go "linenos=inline" >}}
kubectl get pod -n kube-system -o wide | grep 'kubernetes-dashboard'
{{< /highlight >}}
2. 在节点上查看kubernetes-dashboard容器ID
{{< highlight go "linenos=inline" >}}
docker ps | grep dashboard
{{< /highlight >}}
3. 获取kubernetes-dashboard容器certs所挂载的宿主主机目录
{{< highlight go "linenos=inline" >}}
docker inspect -f {{.Mounts}} 384d9dc0170b
{{< /highlight >}}
4. 以私有证书配置，生成dashboard证书
{{< highlight go "linenos=inline" >}}
openssl genrsa -des3 -passout pass:x -out dashboard.pass.key 2048
openssl rsa -passin pass:x -in dashboard.pass.key -out dashboard.key
openssl req -new -key dashboard.key -out dashboard.csr
openssl x509 -req -sha256 -days 365 -in dashboard.csr -signkey dashboard.key -out dashboard.crt
{{< /highlight >}}
5. 将生成的dashboard.crt  dashboard.key放到certs对应的宿主主机souce目录
{{< highlight go "linenos=inline" >}}
scp dashboard.crt dashboard.key 192.168.20.214:/var/lib/kubelet/pods/94c8c50b-f484-11e8-80e8-000c29c3dca5/volumes/kubernetes.io~secret/kubernetes-dashboard-certs
{{< /highlight >}}
6. 重启kubernetes-dashboard容器
{{< highlight go "linenos=inline" >}}
docker restart 384d9dc0170b
{{< /highlight >}}

## 使用API Server
https://192.168.20.210:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
返回403
原因:
由于kube-apiserver使用了TLS认证，而我们的真实物理机上的浏览器使用匿名证书（因为没有可用的证书）去访问Dashboard，导致授权失败而不无法访问

## 使用kubectl proxy
Master上执行nohup kubecll proxy &，然后使用如下地址访问Dashboard
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
限制就是必须在Master上访问，在主节点上，我们执行nohup kubectl proxy --address=192.168.20.210 --disable-filter=true & 开启代理
address表示外界可以使用192.168.20.210来访问Dashboard，我们也可以使用0.0.0.0
disable-filter=true表示禁用请求过滤功能，否则我们的请求会被拒绝，并提示 Forbidden (403) Unauthorized
也可以指定端口，具体请查看kubectl proxy --help
proxy默认对Master的8001端口进行监听
# 问题
直接实用HTTPS方式请求，浏览器会报错并提示“NET::ERR_CERT_INVALID”
## 方式一: 换火狐浏览器
## 方式二: 启动proxy模式
{{< highlight go "linenos=inline" >}}
kubectl proxy --address=0.0.0.0 --disable-filter=true
{{< /highlight >}}
然后访问
{{< highlight go "linenos=inline" >}}
http://ip:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
{{< /highlight >}}

## 方式三: 生成证书
{{< highlight go "linenos=inline" >}}
mkdir certs
openssl req -nodes -newkey rsa:2048 -keyout certs/dashboard.key -out certs/dashboard.csr -subj "/C=/ST=/L=/O=/OU=/CN=kubernetes-dashboard"
openssl x509 -req -sha256 -days 365 -in certs/dashboard.csr -signkey certs/dashboard.key -out certs/dashboard.crt
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
kubectl create secret generic kubernetes-dashboard-certs --from-file=certs -n kube-system
kubectl create -f kubernetes-dashboard.yaml
{{< /highlight >}}

## 方式三: 取消认证
编辑dashboard的yaml配置文件
{{< highlight go "linenos=inline" >}}
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
{{< /highlight >}}
应用规则
{{< highlight go "linenos=inline" >}}
kubectl apply -f dashboard-admin.yaml
{{< /highlight >}}

右键浏览器快捷方式，在弹出菜单中选择"属性"->"快捷方式"->"目标"，在原有的目标值后面先空格，然后
低版本chrome加上
{{< highlight go "linenos=inline" >}}
--test-type --ignore-certificate-errors
{{< /highlight >}}
高版本chrome加上
{{< highlight go "linenos=inline" >}}
--disable-infobars --ignore-certificate-errors
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

# 权限不够
kubernetes-dashboard-minimal，Role的权限不够，
更改RoleBinding修改为ClusterRoleBinding，并且修改roleRef中的kind和name，使用cluster-admin这个非常牛逼的CusterRole（超级用户权限，其拥有访问kube-apiserver的所有权限）。如下：

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
修改后，重新创建kubernetes-dashboard.yaml，Dashboard就可以拥有访问整个K8S 集群API的权限。我们重新访问Dashboard

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
k8s中的RBAC规定了一个用户或者用户组具有请求哪些api的权限，在配合TLS加密使用的时候，客户端如果想要与apiserver通讯，就必须采用由apiserver CA签发的证书，形成信任关系，建立TLS连接。apiserver通过读取客户端证书的CN字段作为RBAC的用户名，读取O字段作为RBAC的用户组，进而访问服务。

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
## 使用token登陆
* 获取账户kubernetes-dashboard-admin的secret
{{< highlight go "linenos=inline" >}}
kubectl -n kube-system get secret | grep kubernetes-dashboard-admin
{{< /highlight >}}
* 获取secret中的token信息
{{< highlight go "linenos=inline" >}}
kubectl describe secrets $secret -n kube-system | grep token
{{< /highlight >}}


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


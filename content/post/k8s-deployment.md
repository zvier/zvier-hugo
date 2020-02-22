---
title: "k8s deployment"
date: 2020-01-17T17:20:31+08:00
draft: true
categories: ["技术"]
tags: ["k8s"]
---
<!--more-->
# 简述
k8s中，Deployment为Pod和ReplicaSet提供声明式更新，通过在Deployment的yaml文件中声明好目标状态，Deployment Controller就会将Pod和ReplicaSet的实际状态逐渐调和到该目标状态。这样，我们可以定义一个全新的Deployment来创建ReplicaSet，或者删除已有的Deployment创建一个新的替换。  

# 典型Deployment
{{< highlight go >}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
{{< /highlight >}}

# 使用
## 创建Deployment
{{< highlight go "linenos=inline" >}}
kubectl create -f deployment_nginx.yml
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           41m
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-5489c599c4   3         3         3       43m
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5489c599c4-4b2zl   1/1     Running   0          32m
nginx-deployment-5489c599c4-ktlmp   1/1     Running   0          32m
nginx-deployment-5489c599c4-sjphx   1/1     Running   0          32m
{{< /highlight >}}

## 升级Deployment
{{< highlight go "linenos=inline" >}}
kubectl get deployments -o wide
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
nginx-deployment   3/3     3            3           47m   nginx        nginx:1.12.2   app=nginx
{{< /highlight >}}}

{{< highlight go "linenos=inline" >}}
kubectl set image deployment nginx-deployment nginx=nginx:latest
deployment.apps/nginx-deployment image updated
{{< /highlight >}}}

{{< highlight go "linenos=inline" >}}
kubectl get deployments -o wide
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
nginx-deployment   3/3     1            3           49m   nginx        nginx:latest   app=nginx
{{< /highlight >}}}

---
title: "helm"
date: 2020-03-08T13:25:47+08:00
draft: true
categories: ["技术"]
tags: ["helm"]
---
毕竟几人真得鹿，不知终日梦为鱼。（黄庭坚）
<!--more-->
# 简介
helm是k8s的一个包管理工具，类似于centos上的yum，使用helm管理应用的chart包，可以简化k8s应用的部署和管理。以下主要介绍helm3的安装和使用。

# helm基本概念
## Chart
Chart是k8s应用的打包格式，由一系列文件组成，类似rpm包，包含如下内容:

1.Chart.yaml: 用来描述chart的摘要信息  
2.README.md(可选): readme文件  
3.LICENSE(可选): 描述chart的许可信息  
5.values.yaml:chart支持在安装的时候做对配置参数做定制化配置，value.yaml文件为配置参数的默认值  
6.templates目录: 各类k8s资源的配置模板目录  

# 客户端安装
去项目releases页下载相应的helm版本，解压并将helm二进制置于$PATH路径下
{{< highlight go "linenos=inline" >}}
wget https://get.helm.sh/helm-v3.1.1-linux-amd64.tar.gz
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
helm version
version.BuildInfo{Version:"v3.1.1", GitCommit:"afe70585407b420d0097d07b21c47dc511525ac8", GitTreeState:"clean", GoVersion:"go1.13.8"}
{{< /highlight >}}

helm 3.0中，默认没有添加chart仓库，需要手动添加，以下是一些常用charts库:
{{< highlight bash >}}
 helm repo add elastic    https://helm.elastic.co
 helm repo add gitlab     https://charts.gitlab.io
 helm repo add harbor     https://helm.goharbor.io
 helm repo add bitnami    https://charts.bitnami.com/bitnami
 helm repo add incubator  https://kubernetes-charts-incubator.storage.googleapis.com
 helm repo add stable     https://kubernetes-charts.storage.googleapis.com
 helm repo add aliyun     https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
 helm repo add reactiveops-stable https://charts.reactiveops.com/stable
{{< /highlight >}}

helm repo其它操作
{{< highlight bash >}}
helm repo list
helm repo remove incubator
{{< /highlight >}}

添加完仓库后，需要执行更新命令，将仓库的信息进行同步

{{< highlight go "linenos=inline" >}}
helm repo update
{{< /highlight >}}

# 安装应用
1.查询
{{< highlight go "linenos=inline" >}}
helm search repo nginx
helm search repo traefik
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
helm show chart bitnami/nginx
apiVersion: v1
appVersion: 1.16.1
description: Chart for the nginx server
home: http://www.nginx.org
......
{{< /highlight >}}

2.下载
{{< highlight go "linenos=inline" >}}
helm pull bitnami/nginx
nginx-5.1.7.tgz

tar zxvf nginx-5.1.7.tgz
nginx/Chart.yaml
nginx/values.yaml
nginx/templates/NOTES.txt
nginx/templates/_helpers.tpl
nginx/templates/deployment.yaml
nginx/templates/ingress.yaml
nginx/templates/server-block-configmap.yaml
nginx/templates/servicemonitor.yaml
nginx/templates/svc.yaml
nginx/templates/tls-secrets.yaml
nginx/.helmignore
nginx/README.md
nginx/ci/values-with-ingress-metrics-and-serverblock.yaml
nginx/values.schema.json
{{< /highlight >}}

2.安装
helm 3在install时，必须指定release名称或增加--generate-name
{{< highlight go "linenos=inline" >}}
// –namespace：指定安装的namespace
helm install stable/redis --generate-name
helm install nginx bitnami/nginx -n default
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
helm list
NAME            	NAMESPACE	REVISION	UPDATED                               	STATUS  	CHART       	APP VERSION
redis-1583651065	default  	1       	2020-03-08 15:04:29.07808891 +0800 CST	deployed	redis-10.5.6	5.0.7

helm list --all-namespaces
{{< /highlight >}}

3.查看应用状态
{{< highlight go "linenos=inline" >}}
helm status nginx -n qa
{{< /highlight >}}

4.查看全部应用（包含安装和卸载的应用）
{{< highlight go "linenos=inline" >}}
helm list -n qa --all
{{< /highlight >}}


# 卸载应用
卸载应用使用helm uninstall命令。
{{< highlight go "linenos=inline" >}}
helm uninstall nginx -n qa # 卸载，不保留安装记录
helm uninstall nginx -n qa --keep-history # 卸载并保留安装记录
{{< /highlight >}}

# 升级应用
创建新的配置

$ cat > values.yaml << EOF

service.type: NodePort
service.nodePorts.http: 30002

EOF
应用更新

$ helm upgrade -f values.yaml nginx bitnami/nginx -n mydlqcloud
查看新配置是否生效

$ helm get values nginx -n mydlqcloud

USER-SUPPLIED VALUES:
service.nodePorts.http: 30002
service.type: NodePort

应用回滚
如果升级过程发生错误，进行回滚，首先查看应用的历史版本：

$ helm history nginx -n mydlqcloud

REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION
1               Sat Nov 23 17:40:40 2019        superseded      nginx-4.3.13    1.16.1          Install complete
2               Sat Nov 23 17:44:44 2019        deployed        nginx-4.3.13    1.16.1          Upgrade complete
知道 REVISION 号后就可以进行回滚操作：

$ helm rollback nginx 1 -n mydlqcloud

Rollback was a success! Happy Helming!

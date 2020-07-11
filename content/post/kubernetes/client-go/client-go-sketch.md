---
title: "client-go 源码分析一：源码结构"
date: 2020-07-04T20:03:43+08:00
draft: true
categories: ["技术"]
tags: ["k8s"]
---
<!--more-->
# 简述
`k8s.io/client-go`包提供了对api-server服务访问的封装，是一个客户端库，各个组建对api-server的访问都是通过`client-go`进行。因此`client-go`在二次开发中也是相当重要。

# 源码结构
以下是`k8s.io/client-go`包的主要源码目录结构。
{{< highlight go "linenos=inline" >}}
tree k8s.io/client-go/ -L 1
k8s.io/client-go/
├── discovery
├── dynamic
├── examples
├── informers
├── kubernetes
├── listers
├── metadata
├── pkg
├── plugin
├── rest
├── restmapper
├── scale
├── testing
├── third_party
├── tools
├── transport
└── util
{{< /highlight >}}

# 功能说明
| 子目录      | 功能                                                                                                                                                             |
|-------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| discovery   | 提供DiscoveryClient                                                                                                                                              |
| dynamic     | 提供DynamicClient                                                                                                                                                |
| examples    | 开发示例                                                                                                                                                         |
| kubernetes  | 提供ClientSet                                                                                                                                                    |
| listers     | 为每一种资源提供Lister功能，对Get/List请求提供只读的缓存数据                                                                                                     |
| metadata    | 为各种Client提供请求元数据结构                                                                                                                                   |
| pkg         | 提供版本等非业务信息                                                                                                                                             |
| plugin      | 提供Openstack，GCP和Azure等云服务商授权插件                                                                                                                      |
| rest        | 提供RESTClient，对api-server执行RESTful操作                                                                                                                      |
| restmapper  |                                                                                                                                                                  |
| scale       | 提供ScaleClient，用于Deployment、ReplicaSet、Replication Controller等资源                                                                                        |
| testing     |                                                                                                                                                                  |
| third_party |                                                                                                                                                                  |
| tools       | 提供常用工具，比如SharedInformer、Reflector、DealtFIFO、Indexers等小工具，通过它们之间组合成一个小系统，为Client查询提供缓存机制，减少向api-server发起请求的压力 |
| transport   | 提供安全的TCP连接，支持HTTP Stream，比如在客户端和容器之间传输二进制流的exec、attach操作                                                                         |
| util        | 提供WorkQueue、Certificate证书管理等辅助功能                                                                                                                     |

# 核心功能
顾名思义，`k8s.io/client-go`包主要就是提供各种Client，方便客户端与服务端通信，client-go支持四种Client与api-server交互，如下：
![client-go-client.jpg](/img/k8s/client-go/client-go-client.jpg)


## RESTClient
RESTClient是最基础的客户端，通过对HTTP Request的进行封装实现了一套RESTful风格的API

## ClientSet
基于RESTClient封装了对Resource和Version的管理方法。每一个Resource都有一个客户端，ClientSet是这些客户端的集合。这里的Resource是指kubernetes的内置资源。

## DynamicClient
类似ClientSet，不同的是ClientSet仅能访问kubernetes的内置资源，但DynamicClient可以处理kubernetes中的所有资源，包括CRD自定义的资源。

## DiscoveryClient
DiscoveryClient主要用于发现kube-apiserver所支持的资源组(Group)，资源版本(Version)，信资源息(Resource)等。


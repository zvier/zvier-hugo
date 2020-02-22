---
title: "grpc interceptor"
date: 2020-02-10T09:12:36+08:00
draft: true
categories: ["技术"]
tags: ["grpc"]
---
汀洲采白苹，日落江南春。洞庭有归客，潇湘逢故人。故人何不返，春华复应晚。不道新知乐，只言行路远。——南朝.柳恽《江南曲》
<!--more-->
# 简述
如果想在每个RPC方法的前或后做某些事情，那拦截器（interceptor）正是你想要的工具

# 方式
在 gRPC 中，有两大类RPC方法，与拦截器的对应关系是
普通方法：一元拦截器（grpc.UnaryInterceptor）
流方法：流拦截器（grpc.StreamInterceptor）
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}


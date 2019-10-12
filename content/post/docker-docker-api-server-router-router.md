---
title: "docker-docker-api-server-router-router"
date: 2019-10-03T09:47:23+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
# 简述
> 但愿那月落重生灯再红——汤显祖«牡丹亭»
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/docker/docker/api/server/router/router.go
{{< /highlight >}}


# Router
Router定义了一个接口，方法Routes返回Route切片  
{{< highlight go "linenos=inline" >}}
// Router defines an interface to specify a group of routes to add to the docker server.
type Router interface {
	// Routes returns the list of routes to add to the docker server.
	Routes() []Route
}
{{< /highlight >}}

# Route
Route接口约定了docker server的单个API route所具备的方法  
{{< highlight go "linenos=inline" >}}
import "github.com/docker/docker/api/server/httputils"
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// Route defines an individual API route in the docker server.
type Route interface {
	// Handler returns the raw function to create the http handler.
	Handler() httputils.APIFunc
	// Method returns the http method that the route responds to.
	Method() string
	// Path returns the subpath where the route responds to.
	Path() string
}
{{< /highlight >}}
1. API endpoint的签名APIFunc: [httputils.APIFunc](http://www.zvier.top/post/docker-docker-api-server-httputils-httputils/#apifunc)

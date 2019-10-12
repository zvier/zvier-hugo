---
title: "docker-docker-api-server-router_swapper"
date: 2019-10-11T21:30:18+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---

# 简述
桃李春风一杯酒，江湖夜雨十年灯。——黄庭坚《寄黄几复》
<!--more-->

# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/docker/docker/api/server/router_swapper.go
{{< /highlight >}}

# routerSwapper
routerSwapper是对路由器的包装，这里也可以称之为路由交换器，用于路由交换
{{< highlight go "linenos=inline" >}}
import (
	"net/http"
	"sync"
	"github.com/gorilla/mux"
)
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// routerSwapper is an http.Handler that allows you to swap
// mux routers.
type routerSwapper struct {
	mu     sync.Mutex
	router *mux.Router
}
{{< /highlight >}}

# Swap
路由交换  
{{< highlight go "linenos=inline" >}}
// Swap changes the old router with the new one.
func (rs *routerSwapper) Swap(newRouter *mux.Router) {
	rs.mu.Lock()
	rs.router = newRouter
	rs.mu.Unlock()
}
{{< /highlight >}}

# routerSwapper.ServeHTTP
实现http.Handler接口
{{< highlight go "linenos=inline" >}}
// ServeHTTP makes the routerSwapper to implement the http.Handler interface.
func (rs *routerSwapper) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	rs.mu.Lock()
	router := rs.router
	rs.mu.Unlock()
	router.ServeHTTP(w, r)
}
{{< /highlight >}}

## http.Handler
{{< highlight go "linenos=inline" >}}
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
{{< /highlight >}}

A Handler responds to an HTTP request.

ServeHTTP should write reply headers and data to the ResponseWriter and then return. Returning signals that the request is finished; it is not valid to use the ResponseWriter or read from the Request.Body after or concurrently with the completion of the ServeHTTP call.

Depending on the HTTP client software, HTTP protocol version, and any intermediaries between the client and the Go server, it may not be possible to read from the Request.Body after writing to the ResponseWriter. Cautious handlers should read the Request.Body first, and then reply.

Except for reading the body, handlers should not modify the provided Request.

If ServeHTTP panics, the server (the caller of ServeHTTP) assumes that the effect of the panic was isolated to the active request. It recovers the panic, logs a stack trace to the server error log, and either closes the network connection or sends an HTTP/2 RST_STREAM, depending on the HTTP protocol. To abort a handler so the client sees an interrupted response but the server doesn't log an error, panic with the value ErrAbortHandler.

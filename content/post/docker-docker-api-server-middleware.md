---
title: "docker-docker-api-server-middleware"
date: 2019-10-11T21:14:39+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
# 简述
少年听雨歌楼上。红烛昏罗帐。壮年听雨客舟中。江阔云低、断雁叫西风。而今听雨僧庐下。鬓已星星也。悲欢离合总无情。一任阶前、点滴到天明。——蒋捷《虞美人 听雨》

<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/docker/docker/api/server/middleware.go
{{< /highlight >}}

# Server.handlerWithGlobalMiddlewares
{{< highlight go "linenos=inline" >}}
import (
	"github.com/docker/docker/api/server/httputils"
	"github.com/docker/docker/api/server/middleware"
	"github.com/sirupsen/logrus"
)
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// handlerWithGlobalMiddlewares wraps the handler function for a request with
// the server's global middlewares(n. 中间件；中间设备). The order of the middlewares is backwards(倒放,回放),
// meaning that the first in the list will be evaluated last.
func (s *Server) handlerWithGlobalMiddlewares(handler httputils.APIFunc) httputils.APIFunc {
	next := handler

	for _, m := range s.middlewares {
		next = m.WrapHandler(next)
	}

	if s.cfg.Logging && logrus.GetLevel() == logrus.DebugLevel {
		next = middleware.DebugRequestMiddleware(next)
	}

	return next
}
{{< /highlight >}}

## 引用说明
1. [httputils.APIFunc](http://www.zvier.top/post/docker-docker-api-server-httputils-httputils/#apifunc)
2. [Server](http://www.zvier.top/post/docker-docker-api-server-server/#server)

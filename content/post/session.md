---
title: "session"
date: 2020-02-10T08:05:27+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
山中无甲子，寒尽不知年。——吴承恩《西游记》
<!--more-->
# NewSession
{{< highlight go "linenos=inline" >}}
// github.com/moby/buildkit/session/session.go
// NewSession returns a new long running session
func NewSession(ctx context.Context, name, sharedKey string) (*Session, error) {
    id := identity.NewID()

    serverOpts := []grpc.ServerOption{}
    if span := opentracing.SpanFromContext(ctx); span != nil {
        tracer := span.Tracer()
        serverOpts = []grpc.ServerOption{
            grpc.StreamInterceptor(otgrpc.OpenTracingStreamServerInterceptor(span.Tracer(), traceFilter())),
            grpc.UnaryInterceptor(otgrpc.OpenTracingServerInterceptor(tracer, traceFilter())),
        }
    }

    s := &Session{
        id:         id,
        name:       name,
        sharedKey:  sharedKey,
        grpcServer: grpc.NewServer(serverOpts...),
    }

    grpc_health_v1.RegisterHealthServer(s.grpcServer, health.NewServer())

    return s, nil
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

# 引用
[dentity.NewID](http://www.zvier.top/post/identity/#newid)

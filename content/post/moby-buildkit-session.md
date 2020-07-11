---
title: "moby buildkit session"
date: 2020-02-29T21:40:15+08:00
draft: true
categories: ["技术"]
tags: ["buildkit"]
---
恨君不似江楼月，南北东西，南北东西，只有相随无别离。恨君却似江楼月，暂满还亏，暂满还亏，待得团圆是几时？——吕本中《采桑子》
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/moby/buildkit/session/session.go
{{< /highlight >}}

# 常量
{{< highlight go "linenos=inline" >}}
const (
	headerSessionID        = "X-Docker-Expose-Session-Uuid"
	headerSessionName      = "X-Docker-Expose-Session-Name"
	headerSessionSharedKey = "X-Docker-Expose-Session-Sharedkey"
	headerSessionMethod    = "X-Docker-Expose-Session-Grpc-Method"
)
{{< /highlight >}}

# Dialer
拨号器Dialer是一个函数类型，该类型实例返回一个网络连接: net.Conn，session
将会用到它
{{< highlight go "linenos=inline" >}}
// Dialer returns a connection that can be used by the session
type Dialer func(ctx context.Context, proto string, meta map[string][]string) (net.Conn, error)
{{< /highlight >}}

# Attachable
Attachable是session包暴露出的一个接口类型，用于注册grpc.Server
{{< highlight go "linenos=inline" >}}
// Attachable defines a feature that can be exposed on a session
type Attachable interface {
	Register(*grpc.Server)
}
{{< /highlight >}}

# Session
Session定义了client和daemon端之间的一个长连接，上面拨号器返回的net.Conn将塞到Session结构中
{{< highlight go "linenos=inline" >}}
// Session is a long running connection between client and a daemon
type Session struct {
	id         string
	name       string
	sharedKey  string
	ctx        context.Context
	cancelCtx  func()
	done       chan struct{}
	grpcServer *grpc.Server
	conn       net.Conn
}
{{< /highlight >}}

# NewSession
NewSession返回一个Session实例
{{< highlight go "linenos=inline" >}}
// NewSession returns a new long running session
func NewSession(ctx context.Context, name, sharedKey string) (*Session, error) {
    // 生产全局唯一的session id
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

## grpc.Server
Server is a gRPC server to serve RPC requests.
{{< highlight go "linenos=inline" >}}
// import "google.golang.org/grpc"
type Server struct {
    // contains filtered or unexported fields
}
{{< /highlight >}}

## grpc.NewServer
NewServer creates a gRPC server which has no service registered and has not started to accept requests yet.
{{< highlight go "linenos=inline" >}}
func NewServer(opt ...ServerOption) *Server
{{< /highlight >}}

## grpc_health_v1.RegisterHealthServer
{{< highlight go "linenos=inline" >}}
// import "google.golang.org/grpc/health/grpc_health_v1"
func RegisterHealthServer(s *grpc.Server, srv HealthServer)
{{< /highlight >}}

## grpc_health_v1.HealthServer
{{< highlight go "linenos=inline" >}}
// import "google.golang.org/grpc/health/grpc_health_v1"
type HealthServer interface {
    Check(context.Context, *HealthCheckRequest) (*HealthCheckResponse, error)
    Watch(*HealthCheckRequest, Health_WatchServer) error
}
{{< /highlight >}}

# Session.Allow
Session.Allow使得Session中的grpcServer可被访问
{{< highlight go "linenos=inline" >}}
// Allow enables a given service to be reachable through the grpc session
func (s *Session) Allow(a Attachable) {
	a.Register(s.grpcServer)
}
{{< /highlight >}}

# Session.ID
返回Session的id
{{< highlight go "linenos=inline" >}}
// ID returns unique identifier for the session
func (s *Session) ID() string {
	return s.id
}
{{< /highlight >}}

# Session.Run
启动Session
{{< highlight go "linenos=inline" >}}
// Run activates the session
func (s *Session) Run(ctx context.Context, dialer Dialer) error {
	ctx, cancel := context.WithCancel(ctx)
	s.cancelCtx = cancel
	s.done = make(chan struct{})

	defer cancel()
	defer close(s.done)

	meta := make(map[string][]string)
	meta[headerSessionID] = []string{s.id}
	meta[headerSessionName] = []string{s.name}
	meta[headerSessionSharedKey] = []string{s.sharedKey}

	for name, svc := range s.grpcServer.GetServiceInfo() {
		for _, method := range svc.Methods {
			meta[headerSessionMethod] = append(meta[headerSessionMethod], MethodURL(name, method.Name))
		}
	}
	conn, err := dialer(ctx, "h2c", meta)
	if err != nil {
		return errors.Wrap(err, "failed to dial gRPC")
	}
	s.conn = conn
	serve(ctx, s.grpcServer, conn)
	return nil
}
{{< /highlight >}}

## Server.GetServiceInfo
GetServiceInfo returns a map from service names to ServiceInfo. Service names include the package names, in the form of <package>.<service>.
{{< highlight go "linenos=inline" >}}
// import "google.golang.org/grpc"
func (s *Server) GetServiceInfo() map[string]ServiceInfo
{{< /highlight >}}

## grpc.ServiceInfo
ServiceInfo contains unary RPC method info, streaming RPC method info and metadata for a service.
{{< highlight go "linenos=inline" >}}
// import "google.golang.org/grpc"
type ServiceInfo struct {
    Methods []MethodInfo
    // Metadata is the metadata specified in ServiceDesc when registering service.
    Metadata interface{}
}
{{< /highlight >}}


# Session.Close
关闭Session
{{< highlight go "linenos=inline" >}}
// Close closes the session
func (s *Session) Close() error {
	if s.cancelCtx != nil && s.done != nil {
		if s.conn != nil {
			s.conn.Close()
		}
		s.grpcServer.Stop()
		<-s.done
	}
	return nil
}
{{< /highlight >}}

## Server.Stop
Stop stops the gRPC server. It immediately closes all open connections and listeners. It cancels all active RPCs on the server side and the corresponding pending RPCs on the client side will get notified by connection errors.
{{< highlight go "linenos=inline" >}}
// import "google.golang.org/grpc"
func (s *Server) Stop()
{{< /highlight >}}


# Session.context
返回Session的上下文
{{< highlight go "linenos=inline" >}}
func (s *Session) context() context.Context {
	return s.ctx
}
{{< /highlight >}}

# Session.closed
判断Session是否已经关闭
{{< highlight go "linenos=inline" >}}
func (s *Session) closed() bool {
	select {
	case <-s.context().Done():
		return true
	default:
		return false
	}
}
{{< /highlight >}}

# MethodURL
将service和method拼接成一个gRPC URL
{{< highlight go "linenos=inline" >}}
// MethodURL returns a gRPC method URL for service and method name
func MethodURL(s, m string) string {
	return "/" + s + "/" + m
}
{{< /highlight >}}

# traceFilter
{{< highlight go "linenos=inline" >}}
func traceFilter() otgrpc.Option {
	return otgrpc.IncludingSpans(func(parentSpanCtx opentracing.SpanContext,
		method string,
		req, resp interface{}) bool {
		return !strings.HasSuffix(method, "Health/Check")
	})
}
{{< /highlight >}}

## otgrpc.Option
Option instances may be used in OpenTracing(Server|Client)Interceptor initialization.
See this post about the "functional options" pattern: http://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis
{{< highlight go "linenos=inline" >}}
// import "github.com/grpc-ecosystem/grpc-opentracing/go/otgrpc"
type Option func(o *options)
{{< /highlight >}}

## otgrpc.IncludingSpans
IncludingSpans binds a IncludeSpanFunc to the options
{{< highlight go "linenos=inline" >}}
// import "github.com/grpc-ecosystem/grpc-opentracing/go/otgrpc"
func IncludingSpans(inclusionFunc SpanInclusionFunc) Option
{{< /highlight >}}

## otgrpc.SpanInclusionFunc
SpanInclusionFunc provides an optional mechanism to decide whether or not to trace a given gRPC call. Return true to create a Span and initiate tracing, false to not create a Span and not trace.

parentSpanCtx may be nil if no parent could be extraction from either the Go context.Context (on the client) or the RPC (on the server).
{{< highlight go "linenos=inline" >}}
type SpanInclusionFunc func(
    parentSpanCtx opentracing.SpanContext,
    method string,
    req, resp interface{}) bool
{{< /highlight >}}

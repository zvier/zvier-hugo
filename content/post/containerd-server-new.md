---
title: "containerd源码解析(3)——containerd server初始化"
date: 2019-09-01T17:34:05+08:00
draft: true
categories: [""]
tags: ["containerd"]
---
# 简述
在containerd
app的Action中，有New了一个server，server.New根据config会创建和初始化一个containerd
server，所以也非常重要。
{{< highlight go "linenos=inline" >}}
server, err := server.New(ctx, config)
{{< /highlight >}}
以下就是server.New函数的签名
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/services/server/server.go
// New creates and initializes a new containerd server
func New(ctx context.Context, config *srvconfig.Config) (*Server, error)
{{< /highlight >}}

## apply
server.New首先是通过调用apply函数将config配置应用到server上，比如sys.SetOOMScore，cgroups.Load等  
{{< highlight go "linenos=inline" >}}
func New(ctx context.Context, config *srvconfig.Config) (*Server, error) {
    if err := apply(ctx, config); err != nil {
        return nil, err
    }
    ...
}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/services/server/server_linux.go
// apply sets config settings on the server process
func apply(ctx context.Context, config *srvconfig.Config) error {
    if config.OOMScore != 0 {
        log.G(ctx).Debugf("changing OOM score to %d", config.OOMScore)
        if err := sys.SetOOMScore(os.Getpid(), config.OOMScore); err != nil {
            log.G(ctx).WithError(err).Errorf("failed to change OOM score to %d", config.OOMScore)
        }
    }
    if config.Cgroup.Path != "" {
        cg, err := cgroups.Load(cgroups.V1, cgroups.StaticPath(config.Cgroup.Path))
        if err != nil {
            if err != cgroups.ErrCgroupDeleted {
                return err
            }
            if cg, err = cgroups.New(cgroups.V1, cgroups.StaticPath(config.Cgroup.Path), &specs.LinuxResources{}); err != nil {
                return err
            }
        }
        if err := cg.Add(cgroups.Process{
            Pid: os.Getpid(),
        }); err != nil {
            return err
        }
    }
    return nil
}
{{< /highlight >}}

### sys.SetOOMScore
sys.SetOOMScore主要是设置/proc/$pid/oom_score_adj文件的值，
{{< highlight go "linenos=inline" >}}
// SetOOMScore sets the oom score for the provided pid
func SetOOMScore(pid, score int) error
{{< /highlight >}}
oom-killer会通过比较每个进程的oom_score来挑选要出局的进程，数值越大就越容易被选中，手工调整oom_score是通过oom_score_adj来实现的，比如：
{{< highlight bash "linenos=inline" >}}
echo "-100" > /proc/$pid/oom_score_adj，oom_score_adj可以设置-1000~1000之间的值。设置为-1000时，该进程就被排除在OOM Killer强制终止的对象外。
{{< /highlight >}}

##  LoadPlugins
{{< highlight go "linenos=inline" >}}
func New(ctx context.Context, config *srvconfig.Config) (*Server, error) {
    ...
    plugins, err := LoadPlugins(ctx, config)
    if err != nil {
        return nil, err
    }
    ...
}
{{< /highlight >}}
LoadPlugins会根据config加载所有需要的插件，涉及内容也比较多，后面会单独介绍。我们在启动containerd时，日志里可以看到如下内容：
{{< highlight bash "linenos=inline" >}}
INFO[2019-09] loading plugin "io.containerd.differ.v1.walking"...  type=io.containerd.differ.v1
INFO[2019-09] loading plugin "io.containerd.gc.v1.scheduler"...  type=io.containerd.gc.v1
INFO[2019-09] loading plugin "io.containerd.service.v1.containers-service"...  type=io.containerd.service.v1
INFO[2019-09] loading plugin "io.containerd.service.v1.content-service"...  type=io.containerd.service.v1
INFO[2019-09] loading plugin "io.containerd.service.v1.diff-service"...  type=io.containerd.service.v1
INFO[2019-09] loading plugin "io.containerd.service.v1.images-service"...  type=io.containerd.service.v1
INFO[2019-09] loading plugin "io.containerd.service.v1.leases-service"...  type=io.containerd.service.v1
INFO[2019-09] loading plugin "io.containerd.service.v1.namespaces-service"...  type=io.containerd.service.v1
...
{{< /highlight >}}
这正是LoadPlugins在加载插件

## serverOpts
{{< highlight bash "linenos=inline" >}}
// import "grpc_prometheus "github.com/grpc-ecosystem/go-grpc-prometheus""

func New(ctx context.Context, config *srvconfig.Config) (*Server, error) {
    ...
    serverOpts := []grpc.ServerOption{
        grpc.UnaryInterceptor(grpc_prometheus.UnaryServerInterceptor),
        grpc.StreamInterceptor(grpc_prometheus.StreamServerInterceptor),
    }
    if config.GRPC.MaxRecvMsgSize > 0 {
        serverOpts = append(serverOpts, grpc.MaxRecvMsgSize(config.GRPC.MaxRecvMsgSize))
    }
    if config.GRPC.MaxSendMsgSize > 0 {
        serverOpts = append(serverOpts, grpc.MaxSendMsgSize(config.GRPC.MaxSendMsgSize))
    }
    ...
}
{{< /highlight >}}
### grpc.ServerOption
grpc包中的ServerOption主要用于设置一些诸如credentials,
codec和keepalive等相关选项参数。它是一个interface  
{{< highlight bash "linenos=inline" >}}
// import "google.golang.org/grpc"
type ServerOption interface {
    // contains filtered or unexported methods
}
{{< /highlight >}}

grpc.UnaryInterceptor和grpc.StreamInterceptor这两个函数都用于生成一个ServerOption

### grpc.UnaryInterceptor
UnaryInterceptor returns a ServerOption that sets the UnaryServerInterceptor for the server. Only one unary interceptor can be installed. The construction of multiple interceptors (e.g., chaining) can be implemented at the caller.
{{< highlight bash "linenos=inline" >}}
func UnaryInterceptor(i UnaryServerInterceptor) ServerOption
{{< /highlight >}}

### grpc.UnaryServerInterceptor
UnaryServerInterceptor provides a hook to intercept the execution of a unary RPC on the server. info contains all the information of this RPC the interceptor can operate on. And handler is the wrapper of the service method implementation. It is the responsibility of the interceptor to invoke handler to complete the RPC.
{{< highlight bash "linenos=inline" >}}
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
{{< /highlight >}}

### grpc.StreamInterceptor
StreamServerInterceptor provides a hook to intercept the execution of a streaming RPC on the server. info contains all the information of this RPC the interceptor can operate on. And handler is the service method implementation. It is the responsibility of the interceptor to invoke handler to complete the RPC.
{{< highlight go "linenos=inline" >}}
type StreamServerInterceptor func(srv interface{}, ss ServerStream, info *StreamServerInfo, handler StreamHandler) error
{{< /highlight >}}

### grpc.StreamInterceptor
{{< highlight bash "linenos=inline" >}}
type StreamServerInterceptor func(srv interface{}, ss ServerStream, info *StreamServerInfo, handler StreamHandler) error
{{< /highlight >}}

### grpc.MaxRecvMsgSize
MaxRecvMsgSize returns a ServerOption to set the max message size in bytes the server can receive. If this is not set, gRPC uses the default 4MB.
{{< highlight go "linenos=inline" >}}
func MaxRecvMsgSize(m int) ServerOption
{{< /highlight >}}

### grpc.MaxSendMsgSize
MaxSendMsgSize returns a ServerOption to set the max message size in bytes the server can send. If this is not set, gRPC uses the default `math.MaxInt32`.
{{< highlight go "linenos=inline" >}}
func MaxSendMsgSize(m int) ServerOption
{{< /highlight >}}

## newTTRPCServer
{{< highlight go "linenos=inline" >}}
func New(ctx context.Context, config *srvconfig.Config) (*Server, error) {
    ...
    ttrpcServer, err := newTTRPCServer()
    if err != nil {
        return nil, err
    }
    ...
}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
import "github.com/containerd/ttrpc"
func newTTRPCServer() (*ttrpc.Server, error) {
    return ttrpc.NewServer(ttrpc.WithServerHandshaker(ttrpc.UnixSocketRequireSameUser()))
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
import "google.golang.org/grpc/credentials"
func New(ctx context.Context, config *srvconfig.Config) (*Server, error) {
    ...
    tcpServerOpts := serverOpts
    if config.GRPC.TCPTLSCert != "" {
        log.G(ctx).Info("setting up tls on tcp GRPC services...")
        creds, err := credentials.NewServerTLSFromFile(config.GRPC.TCPTLSCert, config.GRPC.TCPTLSKey)
        if err != nil {
            return nil, err
        }
        tcpServerOpts = append(tcpServerOpts, grpc.Creds(creds))
    }
    ...
}
{{< /highlight >}}

## Server
{{< highlight go "linenos=inline" >}}
func New(ctx context.Context, config *srvconfig.Config) (*Server, error) {
    ...
    var (
         grpcServer = grpc.NewServer(serverOpts...)
         tcpServer  = grpc.NewServer(tcpServerOpts...)

         grpcServices  []plugin.Service
         tcpServices   []plugin.TCPService
         ttrpcServices []plugin.TTRPCService

         s = &Server{
             grpcServer:  grpcServer,
             tcpServer:   tcpServer,
             ttrpcServer: ttrpcServer,
             events:      exchange.NewExchange(),
             config:      config,
         }
         initialized = plugin.NewPluginSet()
         required    = make(map[string]struct{})
     )
     ...
}
{{< /highlight >}}
### grpc.NewServer
NewServer creates a gRPC server which has no service registered and has not started to accept requests yet.
{{< highlight go "linenos=inline" >}}
func NewServer(opt ...ServerOption) *Server
{{< /highlight >}}
Server is a gRPC server to serve RPC requests.
{{< highlight go "linenos=inline" >}}
type Server struct {
    // contains filtered or unexported fields
}
{{< /highlight >}}
### server.Server
{{< highlight go "linenos=inline" >}}
import (
    "github.com/containerd/containerd/events/exchange"
    "github.com/containerd/containerd/plugin"
    "github.com/containerd/ttrpc"
    srvconfig "github.com/containerd/containerd/services/server/config"
    "google.golang.org/grpc"
)
// Server is the containerd main daemon
type Server struct {
    grpcServer  *grpc.Server
    ttrpcServer *ttrpc.Server
    tcpServer   *grpc.Server
    events      *exchange.Exchange
    config      *srvconfig.Config
    plugins     []*plugin.Plugin
}
{{< /highlight >}}

## 
{{< highlight go "linenos=inline" >}}
func New(ctx context.Context, config *srvconfig.Config) (*Server, error) {

     // 遍历config.RequiredPlugins并标记
     for _, r := range config.RequiredPlugins {
         required[r] = struct{}{}
     }
     // 遍历plugins，并处理每一个plugin
     for _, p := range plugins {
         id := p.URI()
         reqID := id
         if config.GetVersion() == 1 {
             reqID = p.ID
         }
         log.G(ctx).WithField("type", p.Type).Infof("loading plugin %q...", id)

         // 初始化plugin的context
         initContext := plugin.NewContext(
             ctx,
             p,
             initialized,
             config.Root,
             config.State,
         )
         initContext.Events = s.events
         initContext.Address = config.GRPC.Address

         // 加载plugin指定的配置
         // load the plugin specific configuration if it is provided
         if p.Config != nil {
             pc, err := config.Decode(p)
             if err != nil {
                 return nil, err
          }
             initContext.Config = pc
         }
         result := p.Init(initContext)
         if err := initialized.Add(result); err != nil {
             return nil, errors.Wrapf(err, "could not add plugin result to plugin set")
         }

         instance, err := result.Instance()
         if err != nil {
             if plugin.IsSkipPlugin(err) {
                 log.G(ctx).WithError(err).WithField("type", p.Type).Infof("skip loading plugin %q...", id)
             } else {
                 log.G(ctx).WithError(err).Warnf("failed to load plugin %s", id)
             }
             if _, ok := required[reqID]; ok {
                 return nil, errors.Wrapf(err, "load required plugin %s", id)
             }
             continue
         }
         delete(required, reqID)

         // 通过判断instance的类型，是plugin.Service，plugin.TTRPCService还是plugin.TCPService，加入到不同的services切片中
         // check for grpc services that should be registered with the server
         if src, ok := instance.(plugin.Service); ok {
             grpcServices = append(grpcServices, src)
         }
         if src, ok := instance.(plugin.TTRPCService); ok {
             ttrpcServices = append(ttrpcServices, src)
         }
         if service, ok := instance.(plugin.TCPService); ok {
             tcpServices = append(tcpServices, service)
         }

         s.plugins = append(s.plugins, result)
     }
     if len(required) != 0 {
         var missing []string
         for id := range required {
             missing = append(missing, id)
         }
         return nil, errors.Errorf("required plugin %s not included", missing)
     }
     ...
}
{{< /highlight >}}
## Register
当所有的plugin初始化后，需要注册到对应的server中
{{< highlight go "linenos=inline" >}}
func New(ctx context.Context, config *srvconfig.Config) (*Server, error) {
    ...
    // register services after all plugins have been initialized
    for _, service := range grpcServices {
        if err := service.Register(grpcServer); err != nil {
            return nil, err
        }
    }
    for _, service := range ttrpcServices {
        if err := service.RegisterTTRPC(ttrpcServer); err != nil {
            return nil, err
        }
    }
    for _, service := range tcpServices {
        if err := service.RegisterTCP(tcpServer); err != nil {
            return nil, err
        }
    }
    return s, nil
}
{{< /highlight >}}


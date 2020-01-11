---
title: "ctr version"
date: 2019-12-29T09:34:04+08:00
draft: true
categories: ["技术"]
tags: ["contaienrd"]
---
钟鼎山林都是梦，人间宠辱休惊。只消闲处遇平生。——辛弃疾《临江仙·钟鼎山林都是梦》
<!--more-->

# 简述
<code>ctr version</code>命令用于查询客户端/服务端版本信息，最简单。通过阅读该部分代码，可以快速熟悉其它功能流程。

# 客户端Command
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/cmd/ctr/commands/version/version.go
package version

import (
	"fmt"
	"os"

	"github.com/containerd/containerd/cmd/ctr/commands"
	"github.com/containerd/containerd/version"
	"github.com/urfave/cli"
)

// Command is a cli command to output the client and containerd server version
var Command = cli.Command{
	Name:  "version",
	Usage: "print the client and server versions",
	Action: func(context *cli.Context) error {
		fmt.Println("Client:")
		fmt.Println("  Version: ", version.Version)
		fmt.Println("  Revision:", version.Revision)
		fmt.Println("  Go version:", version.GoVersion)
		fmt.Println("")
		client, ctx, cancel, err := commands.NewClient(context)
		if err != nil {
			return err
		}
		defer cancel()
		v, err := client.Version(ctx)
		if err != nil {
			return err
		}
		fmt.Println("Server:")
		fmt.Println("  Version: ", v.Version)
		fmt.Println("  Revision:", v.Revision)
		di, err := client.Server(ctx)
		if err != nil {
			return err
		}
		fmt.Println("  UUID:", di.UUID)
		if v.Version != version.Version {
			fmt.Fprintln(os.Stderr, "WARNING: version mismatch")
		}
		if v.Revision != version.Revision {
			fmt.Fprintln(os.Stderr, "WARNING: revision mismatch")
		}
		return nil
	},
}
{{< /highlight >}}

# 客户端Version结构和函数调用
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/client.go

// VersionService returns the underlying VersionClient
func (c *Client) VersionService() versionservice.VersionClient {
    c.connMu.Lock()
    defer c.connMu.Unlock()
    return versionservice.NewVersionClient(c.conn)
}

// Version of containerd
type Version struct {
    // Version number
    Version string
    // Revision from git that was built
    Revision string
}

// Version returns the version of containerd that the client is connected to
func (c *Client) Version(ctx context.Context) (Version, error) {
    c.connMu.Lock()
    if c.conn == nil {
        c.connMu.Unlock()
        return Version{}, errors.Wrap(errdefs.ErrUnavailable, "no grpc connection available")
    }
    c.connMu.Unlock()
    response, err := c.VersionService().Version(ctx, &ptypes.Empty{})
    if err != nil {
        return Version{}, err
    }
    return Version{
        Version:  response.Version,
        Revision: response.Revision,
    }, nil
}
{{< /highlight >}}

# GRPC客户端VersionClient
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/api/services/version/v1/version.pb.go
// VersionClient is the client API for Version service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://godoc.org/google.golang.org/grpc#ClientConn.NewStream.
type VersionClient interface {
    Version(ctx context.Context, in *types.Empty, opts ...grpc.CallOption) (*VersionResponse, error)
}

type versionClient struct {
    cc *grpc.ClientConn
}

func NewVersionClient(cc *grpc.ClientConn) VersionClient {
    return &versionClient{cc}
}

func (c *versionClient) Version(ctx context.Context, in *types.Empty, opts ...grpc.CallOption) (*VersionResponse, error) {
    out := new(VersionResponse)
    err := c.cc.Invoke(ctx, "/containerd.services.version.v1.Version/Version", in, out, opts...)
    if err != nil {
        return nil, err
    }
    return out, nil
}
{{< /highlight >}}

# GRPC服务端
{{< highlight go "linenos=inline" >}}
 // github.com/containerd/containerd/api/services/version/v1/version.pb.go
 // VersionServer is the server API for Version service.
 type VersionServer interface {
     Version(context.Context, *types.Empty) (*VersionResponse, error)
 }
 
 func RegisterVersionServer(s *grpc.Server, srv VersionServer) {
     s.RegisterService(&_Version_serviceDesc, srv)
 }
 
 func _Version_Version_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
     in := new(types.Empty)
     if err := dec(in); err != nil {
         return nil, err
     }
     if interceptor == nil {
         return srv.(VersionServer).Version(ctx, in)
     }
     info := &grpc.UnaryServerInfo{
         Server:     srv,
         FullMethod: "/containerd.services.version.v1.Version/Version",
     }
     handler := func(ctx context.Context, req interface{}) (interface{}, error) {
         return srv.(VersionServer).Version(ctx, req.(*types.Empty))
     }
     return interceptor(ctx, in, info, handler)
 }
 
 var _Version_serviceDesc = grpc.ServiceDesc{
     ServiceName: "containerd.services.version.v1.Version",
     HandlerType: (*VersionServer)(nil),
     Methods: []grpc.MethodDesc{
         {
             MethodName: "Version",
             Handler:    _Version_Version_Handler,
         },
     },
     Streams:  []grpc.StreamDesc{},
     Metadata: "github.com/containerd/containerd/api/services/version/v1/version.proto",
 }
{{< /highlight >}} 

# 服务端实现

{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/services/version
package version

import (
	"context"

	api "github.com/containerd/containerd/api/services/version/v1"
	"github.com/containerd/containerd/plugin"
	ctrdversion "github.com/containerd/containerd/version"
	ptypes "github.com/gogo/protobuf/types"
	"google.golang.org/grpc"
)

var _ api.VersionServer = &service{}

func init() {
	plugin.Register(&plugin.Registration{
		Type:   plugin.GRPCPlugin,
		ID:     "version",
		InitFn: initFunc,
	})
}

func initFunc(ic *plugin.InitContext) (interface{}, error) {
	return &service{}, nil
}

type service struct {
}

func (s *service) Register(server *grpc.Server) error {
	api.RegisterVersionServer(server, s)
	return nil
}

func (s *service) Version(ctx context.Context, _ *ptypes.Empty) (*api.VersionResponse, error) {
	return &api.VersionResponse{
		Version:  ctrdversion.Version,
		Revision: ctrdversion.Revision,
	}, nil
}
{{< /highlight >}}

# 插件注册
每一个Service都作为插件注册
{{< highlight go "linenos=inline" >}}
//github.com/containerd/containerd/plugin
// Register allows plugins to register
func Register(r *Registration) {
    register.Lock()
    defer register.Unlock()

    if r.Type == "" {
        panic(ErrNoType)
    }
    if r.ID == "" {
        panic(ErrNoPluginID)
    }
    if err := checkUnique(r); err != nil {
        panic(err)
    }

    var last bool
    for _, requires := range r.Requires {
        if requires == "*" {
            last = true
        }
    }
    if last && len(r.Requires) != 1 {
        panic(ErrInvalidRequires)
    }

    register.r = append(register.r, r)
}
{{< /highlight >}}

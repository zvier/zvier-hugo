---
title: "moby buildkit api services control control"
date: 2020-03-01T20:17:23+08:00
draft: true
categories: ["技术"]
tags: ["buildkit"]
---
我住长江头，君主长江尾。日日思君不见君。共饮长江水。此水几时休，此恨何时已。只愿君心似我心，定不负、相思意。——李之仪《卜算子》
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/moby/buildkit/api/services/control/control.pb.go
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// ControlClient is the client API for Control service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://godoc.org/google.golang.org/grpc#ClientConn.NewStream.
type ControlClient interface {
    DiskUsage(ctx context.Context, in *DiskUsageRequest, opts ...grpc.CallOption) (*DiskUsageResponse, error)
    Prune(ctx context.Context, in *PruneRequest, opts ...grpc.CallOption) (Control_PruneClient, error)
    Solve(ctx context.Context, in *SolveRequest, opts ...grpc.CallOption) (*SolveResponse, error)
    Status(ctx context.Context, in *StatusRequest, opts ...grpc.CallOption) (Control_StatusClient, error)
    Session(ctx context.Context, opts ...grpc.CallOption) (Control_SessionClient, error)
    ListWorkers(ctx context.Context, in *ListWorkersRequest, opts ...grpc.CallOption) (*ListWorkersResponse, error)
}

type controlClient struct {
    cc *grpc.ClientConn
}

func NewControlClient(cc *grpc.ClientConn) ControlClient {
    return &controlClient{cc}
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
func (c *controlClient) Solve(ctx context.Context, in *SolveRequest, opts ...grpc.CallOption) (*SolveResponse, error) {
    out := new(SolveResponse)
    err := c.cc.Invoke(ctx, "/moby.buildkit.v1.Control/Solve", in, out, opts...)
    if err != nil {
        return nil, err
    }
    return out, nil
}
{{< /highlight >}}

# controlClient.Status
{{< highlight go "linenos=inline" >}}
func (c *controlClient) Status(ctx context.Context, in *StatusRequest, opts ...grpc.CallOption) (Control_StatusClient, error) {
    stream, err := c.cc.NewStream(ctx, &_Control_serviceDesc.Streams[1], "/moby.buildkit.v1.Control/Status", opts...)
    if err != nil {
        return nil, err
    }
    x := &controlStatusClient{stream}
    if err := x.ClientStream.SendMsg(in); err != nil {
        return nil, err
    }
    if err := x.ClientStream.CloseSend(); err != nil {
        return nil, err
    }
    return x, nil
}
{{< /highlight >}}

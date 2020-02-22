---
title: "otgrpc"
date: 2020-02-10T15:42:15+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
<!--more-->
# 简介
otgrpc包提供了OpenTracing功能，支持任何gRPC客户端和服务端，使得使用OpenTracing更加容易  

# OpenTracingStreamServerInterceptor
{{< highlight go "linenos=inline" >}}
// import "github.com/grpc-ecosystem/grpc-opentracing/go/otgrpc"
func OpenTracingStreamServerInterceptor(tracer opentracing.Tracer, optFuncs ...Option) grpc.StreamServerInterceptor
{{< /highlight >}}

OpenTracingStreamServerInterceptor returns a grpc.StreamServerInterceptor suitable for use in a grpc.NewServer call. The interceptor instruments streaming RPCs by creating a single span to correspond to the lifetime of the RPC's stream.

For example:
{{< highlight go "linenos=inline" >}}
s := grpc.NewServer(
    ...,  // (existing ServerOptions)
    grpc.StreamInterceptor(otgrpc.OpenTracingStreamServerInterceptor(tracer)))
{{< /highlight >}}
All gRPC server spans will look for an OpenTracing SpanContext in the gRPC metadata; if found, the server span will act as the ChildOf that RPC SpanContext.

Root or not, the server Span will be embedded in the context.Context for the application-specific gRPC handler(s) to access.

# OpenTracingServerInterceptor
{{< highlight go "linenos=inline" >}}
// import "github.com/grpc-ecosystem/grpc-opentracing/go/otgrpc"
func OpenTracingServerInterceptor(tracer opentracing.Tracer, optFuncs ...Option) grpc.UnaryServerInterceptor
{{< /highlight >}}
OpenTracingServerInterceptor returns a grpc.UnaryServerInterceptor suitable for use in a grpc.NewServer call.

For example:
{{< highlight go "linenos=inline" >}}
s := grpc.NewServer(
    ...,  // (existing ServerOptions)
    grpc.UnaryInterceptor(otgrpc.OpenTracingServerInterceptor(tracer)))
All gRPC server spans will look for an OpenTracing SpanContext in the gRPC metadata; if found, the server span will act as the ChildOf that RPC SpanContext.

Root or not, the server Span will be embedded in the context.Context for the application-specific gRPC handler(s) to access.

{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// You must have some sort of OpenTracing Tracer instance on hand.
var tracer opentracing.Tracer = ...
...

// Set up a connection to the server peer.
conn, err := grpc.Dial(
    address,
    ... // other options
    grpc.WithUnaryInterceptor(
        otgrpc.OpenTracingClientInterceptor(tracer)),
    grpc.WithStreamInterceptor(
        otgrpc.OpenTracingStreamClientInterceptor(tracer)))

// All future RPC activity involving `conn` will be automatically traced.
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// You must have some sort of OpenTracing Tracer instance on hand.
var tracer opentracing.Tracer = ...
...

// Initialize the gRPC server.
s := grpc.NewServer(
    ... // other options
    grpc.UnaryInterceptor(
        otgrpc.OpenTracingServerInterceptor(tracer)),
    grpc.StreamInterceptor(
        otgrpc.OpenTracingStreamServerInterceptor(tracer)))

// All future RPC activity involving `s` will be automatically traced.
{{< /highlight >}}

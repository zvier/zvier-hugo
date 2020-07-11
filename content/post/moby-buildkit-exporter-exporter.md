---
title: "moby buildkit exporter exporter"
date: 2020-03-05T22:00:35+08:00
draft: true
categories: ["技术"]
tags: ["buildkit"]
---
花似伊。柳似伊。花柳青春伤别离。低头双泪垂。长江东。长江西。两岸鸳鸯两处飞。相逢知几时。——欧阳修《长相思》
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/moby/buildkit/exporter/exporter.go
{{< /highlight >}}

# Exporter
Exporter接口约定Resolve方法，解析出一个ExporterInstance实例
{{< highlight go "linenos=inline" >}}
type Exporter interface {
	Resolve(context.Context, map[string]string) (ExporterInstance, error)
}
{{< /highlight >}}

# ExporterInstance
ExporterInstance接口约定exporter实例的方法
{{< highlight go "linenos=inline" >}}
type ExporterInstance interface {
	Name() string
	Export(context.Context, Source) (map[string]string, error)
}
{{< /highlight >}}

# Source
{{< highlight go "linenos=inline" >}}
type Source struct {
    // "github.com/moby/buildkit/cache"
	Ref      cache.ImmutableRef
	Refs     map[string]cache.ImmutableRef
	Metadata map[string][]byte
}
{{< /highlight >}}


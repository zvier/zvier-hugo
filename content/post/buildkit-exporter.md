---
title: "buildkit exporter"
date: 2020-02-10T16:10:44+08:00
draft: true
categories: ["技术"]
tags: ["buildkit"]
---
浪花有意千重雪，桃李无言一队春。 ——五代.李煜《渔歌子》
<!--more-->
# Exporter
{{< highlight go "linenos=inline" >}}
package exporter

import (
	"context"

	"github.com/moby/buildkit/cache"
)

type Exporter interface {
	Resolve(context.Context, map[string]string) (ExporterInstance, error)
}
{{< /highlight >}}

# ExporterInstance
{{< highlight go "linenos=inline" >}}
type ExporterInstance interface {
	Name() string
	Export(context.Context, Source) (map[string]string, error)
}

{{< /highlight >}}

# Source
{{< highlight go "linenos=inline" >}}
type Source struct {
	Ref      cache.ImmutableRef
	Refs     map[string]cache.ImmutableRef
	Metadata map[string][]byte
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

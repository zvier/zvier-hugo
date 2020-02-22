---
title: "Buildah Push 2"
date: 2020-02-10T18:29:16+08:00
draft: true
categories: [""]
tags: [""]
---
# 简述
<!--more-->
# 常量变量
{{< highlight go "linenos=inline" >}}
// github.com/containers/buildah/imagebuildah/push.go
const (
    PullIfMissing = buildah.PullIfMissing
    PullAlways    = buildah.PullAlways
    PullIfNewer   = buildah.PullIfNewer
    PullNever     = buildah.PullNever

    Gzip         = archive.Gzip
    Bzip2        = archive.Bzip2
    Xz           = archive.Xz
    Zstd         = archive.Zstd
    Uncompressed = archive.Uncompressed
)
{{< /highlight >}}

# 类型
{{< highlight go "linenos=inline" >}}
// Mount is a mountpoint for the build container.
type Mount specs.Mount
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

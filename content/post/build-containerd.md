---
title: "Build Containerd"
date: 2019-12-29T08:48:39+08:00
draft: true
categories: ["技术"]
tags: ["containerd"]
---
万里归来颜愈少，微笑，笑时犹带岭梅香。试问岭南应不好，却道：此心安处是吾乡。 ——苏轼《定风波》
<!--more-->

# 方式
{{< highlight bash "linenos=inline" >}}
cd github.com/containerd/containerd
make
{{< /highlight >}}

# 问题一
{{< highlight bash "linenos=inline" >}}
No package 'libseccomp' found
{{< /highlight >}}

{{< highlight bash "linenos=inline" >}}
yum install libseccomp-devel
{{< /highlight >}}

# 问题二
{{< highlight bash "linenos=inline" >}}
fatal error: btrfs/ioctl.h: No such file or directory #include <btrfs/ioctl.h>
{{< /highlight >}}

dependencies on libdevmapper & btrfs
{{< highlight bash "linenos=inline" >}}
yum install device-mapper-devel
yum install btrfs-progs-devel
{{< /highlight >}}

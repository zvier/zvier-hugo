---
title: "buildah 使用"
date: 2020-03-07T10:29:59+08:00
draft: true
categories: ["buildah"]
tags: ["技术"]
---
<!--more-->
# buildah bud
镜像制作命令格式:
{{< highlight go "linenos=inline" >}}
buildah build-using-dockerfile [options] [context]
buildah bud [options] [context]
{{< /highlight >}}

# 使用本地Containerfile制作镜像
{{< highlight go "linenos=inline" >}}
buildah bud .

buildah bud -f Containerfile .

cat ~/Dockerfile | buildah bud -f - .

buildah bud -f Dockerfile.simple -f Dockerfile.notsosimple .

buildah bud -t imageName .

buildah bud --tls-verify=true -t imageName -f Dockerfile.simple .

buildah bud --tls-verify=false -t imageName .

buildah bud --runtime-flag log-format=json .

buildah bud -f Containerfile --runtime-flag debug .

buildah bud --authfile /tmp/auths/myauths.json --cert-dir ~/auth --tls-verify=true --creds=username:password -t imageName -f Dockerfile.simple .

buildah bud --memory 40m --cpu-period 10000 --cpu-quota 50000 --ulimit nofile=1024:1028 -t imageName .

buildah bud --security-opt label=level:s0:c100,c200 --cgroup-parent /path/to/cgroup/parent -t imageName .

buildah bud --volume /home/test:/myvol:ro,Z -t imageName .

buildah bud -v /var/lib/dnf:/var/lib/dnf:O -t imageName .

buildah bud --layers -t imageName .

buildah bud --no-cache -t imageName .

buildah bud -f Containerfile --layers --force-rm -t imageName .

buildah bud --no-cache --rm=false -t imageName .

buildah bud --dns-search=example.com --dns=223.5.5.5 --dns-option=use-vc .
{{< /highlight >}}



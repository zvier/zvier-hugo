---
title: "Buildkit Usage"
date: 2020-03-01T08:04:21+08:00
draft: true
categories: ["技术"]
tags: ["buildkit"]
---
雪似梅花，梅花似雪，似和不似都奇绝。——吕本中《踏莎行》
<!--more-->
# 编译
{{< highlight go "linenos=inline" >}}
make
{{< /highlight >}}

# daemon启动
{{< highlight go "linenos=inline" >}}
bin/buildkitd
{{< /highlight >}}

# build
{{< highlight go "linenos=inline" >}}
bin/buildctl build  --frontend=dockerfile.v0 --local context=test --local dockerfile=test/ --output type=local,dest=images/test
{{< /highlight >}}


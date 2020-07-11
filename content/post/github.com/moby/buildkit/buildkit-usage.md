---
title: "buildkit 使用"
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
buildctl build --frontend=dockerfile.v0 --local context=. --local dockerfile=.
buildctl build --frontend=dockerfile.v0 --local context=. --local dockerfile=. --opt target=foo --opt build-arg:foo=bar
{{< /highlight >}}

--local表示将client端本地源文件发送给builder，这些源文件包括context指定的build上下文和dockerfile指定的Dockerfile


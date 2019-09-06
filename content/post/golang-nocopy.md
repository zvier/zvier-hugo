---
title: "Golang Nocopy"
date: 2019-06-24T06:41:12+08:00
draft: true
categories: ["golang"]
tags: [""]
---
# noCopy

{{< highlight go "linenos=inline" >}}
// go/src/sync/cond.go
// noCopy may be embedded into structs which must not be copied
// after the first use.
type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
{{< /highlight >}}

> noCopy is an annotation that the containing type shouldn't be copied by value. It has zero size so may be embedded freely.
noCopy在sync.Pool，sync.Cond，sync.Mutex中都有用到，主要用于禁止值拷贝

# 参考
[runtime: add NoCopy documentation struct type?](https://golang.org/issues/8005#issuecomment-190753527)

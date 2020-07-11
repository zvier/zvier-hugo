---
title: "Go Build Contraint"
date: 2020-06-25T12:04:43+08:00
draft: true
categories: [""]
tags: [""]
---
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

# 构建约束
{{< highlight go "linenos=inline" >}}
// +build linux,!gccgo
{{< /highlight >}}
gc编译器即为Go语言自带的编辑器，而gccgo编译器则为GCC提供的Go语言编译器

构建约束也叫构建标记，是一个开始的行注释，必须出现在文件顶部，紧为了将构建约束与包文档区分开来，构建约束之后跟一个空行。

构建约束空白分割构成OR，逗号分隔构成AND，!开头构成否定，例如:
{{< highlight go "linenos=inline" >}}
// +build linux,386 darwin,!cgo
{{< /highlight >}}
等效于
{{< highlight go "linenos=inline" >}}
(linux AND 386) OR (darwin AND (NOT cgo))
{{< /highlight >}}
一个文件可以有多个构建约束，总体约束是个体约束的AND
{{< highlight go "linenos=inline" >}}
// +build linux darwin
// +build 386
{{< /highlight >}}
等效于
{{< highlight go "linenos=inline" >}}
(linux OR darwin) AND 386
{{< /highlight >}}

仅在使用 cgo 时构建文件，并且仅在 Linux 和 OS X 上构建
{{< highlight go "linenos=inline" >}}
// +build linux,cgo darwin,cgo
{{< /highlight >}}

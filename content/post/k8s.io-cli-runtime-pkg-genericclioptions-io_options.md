---
title: "k8s.io-cli-runtime-pkg-genericclioptions-io_options"
date: 2019-09-07T17:55:41+08:00
draft: true
categories: [""]
tags: ["k8s"]
---

# 简述
<!--more-->

# 文件路径
{{< highlight go "linenos=inline" >}}
k8s.io/cli-runtime/pkg/genericclioptions/io_options.go
{{< /highlight >}}

/*
Copyright 2018 The Kubernetes Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package genericclioptions

import (
	"bytes"
	"io"
	"io/ioutil"
)

# IOStreams
IOStreams提供标准输入、输入以及标准错误输出
{{< highlight go "linenos=inline" >}}
// IOStreams provides the standard names for iostreams.  This is useful for embedding and for unit testing.
// Inconsistent and different names make it hard to read and review code
type IOStreams struct {
	// In think, os.Stdin
	In io.Reader
	// Out think, os.Stdout
	Out io.Writer
	// ErrOut think, os.Stderr
	ErrOut io.Writer
}
{{< /highlight >}}

// NewTestIOStreams returns a valid IOStreams and in, out, errout buffers for unit tests
func NewTestIOStreams() (IOStreams, *bytes.Buffer, *bytes.Buffer, *bytes.Buffer) {
	in := &bytes.Buffer{}
	out := &bytes.Buffer{}
	errOut := &bytes.Buffer{}

	return IOStreams{
		In:     in,
		Out:    out,
		ErrOut: errOut,
	}, in, out, errOut
}

// NewTestIOStreamsDiscard returns a valid IOStreams that just discards
func NewTestIOStreamsDiscard() IOStreams {
	in := &bytes.Buffer{}
	return IOStreams{
		In:     in,
		Out:    ioutil.Discard,
		ErrOut: ioutil.Discard,
	}
}


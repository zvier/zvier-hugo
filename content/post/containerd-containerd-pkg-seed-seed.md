---
title: "containerd-containerd-pkg-seed-seed"
date: 2019-09-08T16:20:56+08:00
draft: true
categories: [""]
tags: ["containerd"]
---
# 简述
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/containerd/containerd/pkg/seed/seed.go
{{< /highlight >}}

# WithTimeAndRand
{{< highlight go "linenos=inline" >}}
// WithTimeAndRand seeds the global math rand generator with nanoseconds
// XOR'ed with a crypto component if available for uniqueness.
func WithTimeAndRand() {
	var (
		b [4]byte
		u int64
	)

	tryReadRandom(b[:])

	// Set higher 32 bits, bottom 32 will be set with nanos
	u |= (int64(b[0]) << 56) | (int64(b[1]) << 48) | (int64(b[2]) << 40) | (int64(b[3]) << 32)

	rand.Seed(u ^ time.Now().UnixNano())
}
{{< /highlight >}}


/*
   Copyright The containerd Authors.

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

package seed

import "golang.org/x/sys/unix"

func tryReadRandom(p []byte) {
	// Ignore errors, just decreases uniqueness of seed
	unix.Getrandom(p, unix.GRND_NONBLOCK)
}

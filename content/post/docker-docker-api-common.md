---
title: "docker-docker-api-common"
date: 2019-10-11T21:06:42+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
我亦飘零久，十年来，深恩负尽，死生师友。——顾贞观《金缕曲二首》

# 简述

<!--more-->
daemon和client端的版本信息  

# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/docker/docker/api/common.go
github.com/docker/docker/api/common_unix.go
{{< /highlight >}}

# DefaultVersion
{{< highlight go "linenos=inline" >}}
// Common constants for daemon and client.
const (
	// DefaultVersion of Current REST API
	DefaultVersion = "1.41"

	// NoBaseImageSpecifier is the symbol used by the FROM
	// command to specify that no base image is to be used.
	NoBaseImageSpecifier = "scratch"
)
{{< /highlight >}}

# MinVersion
{{< highlight go "linenos=inline" >}}
// +build !windows

// MinVersion represents Minimum REST API version supported
const MinVersion = "1.12"
{{< /highlight >}}

---
title: "docker-docker-daemon-apparmor_default"
date: 2019-09-28T07:08:43+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
# 简述
apparmor和selinux一样，都可以提供访问控制机制，保障应用程序的安全运行。armor意为盔甲、装甲，言外之意就是为操作系统和应用程序提供保护免受安全威胁。
<!--more-->
apparmor也是linux的一个安全模块，要使用它系统管理员需要将apparmor的安全配置文件与程序关联起来。
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/docker/docker/daemon/apparmor_default.go
{{< /highlight >}}

# 包声明
{{< highlight go "linenos=inline" >}}
// +build linux

package daemon // import "github.com/docker/docker/daemon"
{{< /highlight >}}

## 条件编译
### 编译标签
上面代码中添加的标注<code>// +build linux</code>称之为编译标签(build tag)，它添加在源代码文件顶部。go build在构建一个包的时，通过读取这个包里的每个源文件编译标签来决定该源文件是否参与本次编译。

编译标签的语法规则如下:

1. 编译标签由空格分隔的编译选项(options)以“或”的逻辑关系组成（a build tag is evaluated as the OR of space-separated options）
2. 每个编译选项由逗号分隔的条件项以逻辑“与”的关系组成（ each option evaluates as the AND of its comma-separated terms）
3. 每个条件项的名字可以用字母和数字表示，在前面加!表示否定的意思（each term is an alphanumeric word or, preceded by !, its negation）
4. 编译标签写在文件最顶部，并在后面留一空行，与包声明隔开，否则编译标签将被当作普通注释   

限制源文件只能在 linux/386或者darwin/386平台下编译
{{< highlight go "linenos=inline" >}}
// +build linux darwin
// +build 386
{{< /highlight >}}


go build的-tags标志，是go语言为我们提供的条件编译方式之一。go build -tags=linux 即可生成调用了对应包的可执行文件。

### 文件后缀
go中通过文件后缀的方式也能实现条件编译，go/build可以在不读取源文件的情况下就可以决定哪些文件不需要参加编译。简单来说如果你的源文件包含后缀：_$GOOS.go，那么这个源文件只会在这个平台下编译, _$GOARCH.go也是如此。 这两个后缀可以结合在一起使用，顺序只能为：_$GOOS_$GOARCH.go

例如
{{< highlight go "linenos=inline" >}}
zsys_freebsd_arm.go   // only builds on freebsd/arm systems
daemon_linux.go       // only builds on linux
{{< /highlight >}}
源文件不能只提供条件编译后缀，还必须有文件名，_linux.go、_freebsd_386.go 这两个源文件在所有的平台下都会被忽略掉，因为go/build将会忽略所有以下划线或者点开头的源文件。


# 常量声明
docker-default是运行容器时默认的配置文件，除非通过security-opt选项覆盖
{{< highlight go "linenos=inline" >}}
docker run --rm -it --security-opt apparmor=docker-default hello-world
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// Define constants for native driver
const (
	defaultApparmorProfile = "docker-default"
)
{{< /highlight >}}

# ensureDefaultAppArmorProfile
{{< highlight go "linenos=inline" >}}
import (
	aaprofile "github.com/docker/docker/profiles/apparmor"
	"github.com/opencontainers/runc/libcontainer/apparmor"
)
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
func ensureDefaultAppArmorProfile() error {
	if apparmor.IsEnabled() {
		loaded, err := aaprofile.IsLoaded(defaultApparmorProfile)
		if err != nil {
			return fmt.Errorf("Could not check if %s AppArmor profile was loaded: %s", defaultApparmorProfile, err)
		}

		// Nothing to do.
		if loaded {
			return nil
		}

		// Load the profile.
		if err := aaprofile.InstallDefault(defaultApparmorProfile); err != nil {
			return fmt.Errorf("AppArmor enabled on system but the %s profile could not be loaded: %s", defaultApparmorProfile, err)
		}
	}

	return nil
}
{{< /highlight >}}

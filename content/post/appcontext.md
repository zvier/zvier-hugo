---
title: "appcontext"
date: 2020-02-09T22:05:46+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
两水夹明镜，双桥落彩虹。人烟寒橘柚，秋色老梧桐。——李白《秋登宣城谢眺北楼》
<!--more-->
# Context
Context()调用会返回一个context，同时通过sync.Once的Do方法，开启一个协程监控进程的终止信号，当收到的终止信号超过exitLimit个时，强制终止进程。这个对于命令行工具非常有用
{{< highlight go "linenos=inline" >}}
// github.com/moby/buildkit/util/appcontext/appcontext.go
import (
    "context"
    "os"
    "os/signal"
    "sync"

    "github.com/sirupsen/logrus"
)

var appContextCache context.Context
var appContextOnce sync.Once

// Context returns a static context that reacts to termination signals of the
// running process. Useful in CLI tools.
func Context() context.Context {
    appContextOnce.Do(func() {
        signals := make(chan os.Signal, 2048)
        signal.Notify(signals, terminationSignals...)

        const exitLimit = 3
        retries := 0

        ctx, cancel := context.WithCancel(context.Background())
        appContextCache = ctx

        go func() {
            for {
                <-signals
                cancel()
                retries++
                if retries >= exitLimit {
                    logrus.Errorf("got %d SIGTERM/SIGINTs, forcing shutdown", retries)
                    os.Exit(1)
                }
            }
        }()
    })
    return appContextCache
}
{{< /highlight >}}

# sync.Once
sync.Once能确保实例对象的Do方法在多线程环境只运行一次，即便改变Do方法里的函数执行体，内部通过互斥锁实现，例:
{{< highlight go "linenos=inline" >}}
var printPid sync.Once

func main() {
    listener, err := net.Listen("tcp", "localhost:18888")
    if err != nil {
        log.Fatal(err)
    }
    defer listener.Close()

    for {
        printPid.Do(func() {
            log.Println("process pid:", os.Getpid())
        })

        conn, err := listener.Accept()
        if err != nil {
            log.Println(err)
        }
        log.Println(conn.RemoteAddr(), "success")
        go handleConn(conn)
    }
}
{{< /highlight >}}
sync.Once内部实现
{{< highlight go "linenos=inline" >}}
// Once is an object that will perform exactly one action.
type Once struct {
    m    Mutex
    done uint32
}

func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 1 {
        return
    }
    // Slow-path.
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
{{< /highlight >}}

# terminationSignals
// github.com/moby/buildkit/util/appcontext/appcontext_unix.go 
{{< highlight go "linenos=inline" >}}
// +build !windows

package appcontext

import (
	"os"

	"golang.org/x/sys/unix"
)

var terminationSignals = []os.Signal{unix.SIGTERM, unix.SIGINT}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

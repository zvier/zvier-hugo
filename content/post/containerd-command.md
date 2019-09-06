---
title: "containerd源码解析(2)——"
date: 2019-09-01T07:53:07+08:00
draft: true
categories: ["docker"]
tags: ["containerd"]
---

# 概述
containerd源码解析(1)中提到来main函数中初始化并执行了一个app，那这个app就是在github.com/containerd/containerd/cmd/containerd/command包中实现的，整个app的构建是基于github.com/urfave/cli包，以下挑取重要部分解读  

# init
init主要初始化日志系统，设置cli的VersionPrinter  
{{< highlight go "linenos=inline" >}}
func init() {
    logrus.SetFormatter(&logrus.TextFormatter{
        TimestampFormat: log.RFC3339NanoFixed,
        FullTimestamp:   true,
    })

    // Discard grpc logs so that they don't mess with our stdio
    grpclog.SetLoggerV2(grpclog.NewLoggerV2(ioutil.Discard, ioutil.Discard, ioutil.Discard))

    cli.VersionPrinter = func(c *cli.Context) {
        fmt.Println(c.App.Name, version.Package, c.App.Version, version.Revision)
    }
}
{{< /highlight >}}

>  github.com/urfave/cli.VersionPrinter  
VersionPrinter prints the version for the App

# App
App函数用于生成一个app，定义好app的命令行，命令行flag，命令行的行为，以及子命令行等  
{{< highlight go "linenos=inline" >}}
// App returns a *cli.App instance.
func App() *cli.App {
     app := cli.NewApp()
     app.Name = "containerd"
     app.Version = version.Version
     app.Usage = usage
     app.Flags = []cli.Flag{
         cli.StringFlag{
             Name:  "config,c",
             Usage: "path to the configuration file",
             Value: defaultConfigPath,
         },
         cli.StringFlag{
             Name:  "log-level,l",
             Usage: "set the logging level [trace, debug, info, warn, error, fatal, panic]",
         },
         cli.StringFlag{
             Name:  "address,a",
             Usage: "address for containerd's GRPC server",
         },
         cli.StringFlag{
             Name:  "root",
             Usage: "containerd root directory",
         },
         cli.StringFlag{
             Name:  "state",
             Usage: "containerd state directory",
         },
     }
    app.Flags = append(app.Flags, serviceFlags()...)
       app.Commands = []cli.Command{
        configCommand,
        publishCommand,
        ociHook,
    }
    app.Action = func(context *cli.Context) error {
        ...
    }
    return app
{{< /highlight >}}
其中在linux环境下，serviceFlags返回nil

# app.Action
app.Action里的逻辑就是app.Run要执行的部分，它是一个函数类型，函数签名为
{{< highlight go "linenos=inline" >}}
func(context *cli.Context) error
{{< /highlight >}}

具体到函数的实现比较多，涉及业务逻辑，也最为重要，下面逐一拆解  

## defaultConfig
{{< highlight go "linenos=inline" >}}
"github.com/containerd/containerd/services/server"
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
app.Action = func(context *cli.Context) error {
    var (
        start   = time.Now()
        signals = make(chan os.Signal, 2048)
        serverC = make(chan *server.Server, 1)
        ctx     = gocontext.Background()
        config  = defaultConfig()
    )
    ...
}
{{< /highlight >}}

defaultConfig主要用于获取containerd的默认配置
{{< highlight go "linenos=inline" >}}
"github.com/containerd/containerd/defaults"
"srvconfig "github.com/containerd/containerd/services/server/config"
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/cmd/containerd/command/config_linux.go
func defaultConfig() *srvconfig.Config {
    return &srvconfig.Config{
        Version: 1,
        Root:    defaults.DefaultRootDir,
        State:   defaults.DefaultStateDir,
        GRPC: srvconfig.GRPCConfig{
            Address:        defaults.DefaultAddress,
            MaxRecvMsgSize: defaults.DefaultMaxRecvMsgSize,
            MaxSendMsgSize: defaults.DefaultMaxSendMsgSize,
        },
        DisabledPlugins: []string{},
        RequiredPlugins: []string{},
    }
}
{{< /highlight >}}

## srvconfig.LoadConfig
{{< highlight go "linenos=inline" >}}
app.Action = func(context *cli.Context) error {
    ...
    // 这里主要是加载配置
    if err := srvconfig.LoadConfig(context.GlobalString("config"), config); err != nil && !os.IsNotExist(err) {
        return err
    }
    ...
{{< /highlight >}}

github.com/urfave/cli包中的GlobalString用于查找一个全局的StringFlag并返回其值，如果没有找到返回空字符串  
{{< highlight go "linenos=inline" >}}
func (c *Context) GlobalString(name string) string
{{< /highlight >}}
再来看下srvconfig.LoadConfig的声明
{{< highlight go "linenos=inline" >}}
"srvconfig "github.com/containerd/containerd/services/server/config"
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// LoadConfig loads the containerd server config from the provided path
func LoadConfig(path string, v *Config) error
{{< /highlight >}}
也就是说LoadConfig函数会将containerd启动时--config指定的配置文件(默认值为/etc/containerd/config.toml)内容加载到v中

## applyFlags
{{< highlight go "linenos=inline" >}}
app.Action = func(context *cli.Context) error {
    ...
    // 将命令行中的flag应用到配置中
    // Apply flags to the config
    if err := applyFlags(context, config); err != nil {
        return err
    }
    ...
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
func applyFlags(context *cli.Context, config *srvconfig.Config) error
{{< /highlight >}}
applyFlags要做的事情就是将containerd启动时root，state，address等指定的值也加载到config中，这里命令行中指定的值会覆盖掉配置文件中的值    

## server.CreateTopLevelDirectories
{{< highlight go "linenos=inline" >}}
"github.com/containerd/containerd/services/server"
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
app.Action = func(context *cli.Context) error {
    ...
    // 创建顶层目录
    // Make sure top-level directories are created early.
    if err := server.CreateTopLevelDirectories(config); err != nil {
        return err
    }
    ...
}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/services/server/server.go
// CreateTopLevelDirectories creates the top-level root and state directories.
func CreateTopLevelDirectories(config *srvconfig.Config) error
{{< /highlight >}}
CreateTopLevelDirectories的主要作用就是判断config.Root，config.State等参数设置是否合法，如果合法就创建这样的两个目录   

## registerUnregisterService
registerUnregisterService这个调用仅在windows上有效，在linux上直接返回(false, nil)
{{< highlight go "linenos=inline" >}}
app.Action = func(context *cli.Context) error {
    ...
    // Stop if we are registering or unregistering against Windows SCM.
    stop, err := registerUnregisterService(config.Root)
    if err != nil {
        logrus.Fatal(err)
    }
    if stop {
        return nil
    }
    ...
}
{{< /highlight >}}

## handleSignals
信号处理相关
{{< highlight go "linenos=inline" >}}
app.Action = func(context *cli.Context) error {
    ...
    done := handleSignals(ctx, signals, serverC)
    // start the signal handler as soon as we can to make sure that
    // we don't miss any signals during boot
    signal.Notify(signals, handledSignals...)
    ...
}
{{< /highlight >}}

handleSignals中开了一个协程，采用一个for-select结构监控signals和serverC通道，如果有收到信号，则根据当时的情况做下一步处理  
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/cmd/containerd/command/main_unix.go
func handleSignals(ctx context.Context, signals chan os.Signal, serverC chan *server.Server) chan struct{} {
    done := make(chan struct{}, 1)
    go func() {
        var server *server.Server
        for {
            select {
            case s := <-serverC:
                server = s
            case s := <-signals:
                log.G(ctx).WithField("signal", s).Debug("received signal")
                switch s {
                case unix.SIGUSR1:
                    dumpStacks(true)
                case unix.SIGPIPE:
                    continue
                default:
                    if server == nil {
                        close(done)
                        return
                    }
                    server.Stop()
                    close(done)
                }
            }
        }
    }()
    return done
}
{{< /highlight >}}

## handledSignals
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/cmd/containerd/command/main_unix.go
var handledSignals = []os.Signal{
    unix.SIGTERM,
    unix.SIGINT,
    unix.SIGUSR1,
    unix.SIGPIPE,
}
{{< /highlight >}}

## mount.SetTempMountLocation

{{< highlight go "linenos=inline" >}}
app.Action = func(context *cli.Context) error {
    ...
    // cleanup temp mounts
    if err := mount.SetTempMountLocation(filepath.Join(config.Root, "tmpmounts")); err != nil {
        return errors.Wrap(err, "creating temp mount location")
    }
    ...
}
{{< /highlight >}}

SetTempMountLocation就是在config.Root(默认/var/lib/containerd/)目录下，创建一个tmpmounts目录，并将tempMountLocation置为config.Root/tmpmounts
{{< highlight go "linenos=inline" >}}
"github.com/containerd/containerd/mount"
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// SetTempMountLocation sets the temporary mount location
func SetTempMountLocation(root string) error {
     root, err := filepath.Abs(root)
     if err != nil {
         return err
     }
     if err := os.MkdirAll(root, 0700); err != nil {
         return err
     }
     tempMountLocation = root
     return nil
}
{{< /highlight >}}

## mount.CleanupTempMounts
{{< highlight go "linenos=inline" >}}
app.Action = func(context *cli.Context) error {
    ....
    // unmount all temp mounts on boot for the server
    warnings, err := mount.CleanupTempMounts(0)
    if err != nil {
        log.G(ctx).WithError(err).Error("unmounting temp mounts")
    }
    ...
}
{{< /highlight >}}
CleanupTempMounts主要时读取/proc/self/mountinfo文件，解析出当前所有的mounts，如果这个mount的挂在点前缀为tempMountLocation(/var/lib/containerd/tmpmounts)，那这个mount就需要清理掉它的目录  
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/mount/temp_unix.go
// CleanupTempMounts all temp mounts and remove the directories
func CleanupTempMounts(flags int) (warnings []error, err error)
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
app.Action = func(context *cli.Context) error {
    ...
    for _, w := range warnings {
        log.G(ctx).WithError(w).Warn("cleanup temp mount")
    }
    var (
        address      = config.GRPC.Address
        ttrpcAddress = fmt.Sprintf("%s.ttrpc", config.GRPC.Address)
    )
    if address == "" {
        return errors.New("grpc address cannot be empty")
    }
    log.G(ctx).WithFields(logrus.Fields{
        "version":  version.Version,
        "revision": version.Revision,
    }).Info("starting containerd")
    ...
}
{{< /highlight >}}

## server.New
{{< highlight go "linenos=inline" >}}
app.Action = func(context *cli.Context) error {
    ...
    server, err := server.New(ctx, config)
    if err != nil {
          return err
     }
     ...
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/services/server
// New creates and initializes a new containerd server
func New(ctx context.Context, config *srvconfig.Config) (*Server, error)
{{< /highlight >}}
server.New会根据config创建和初始化一个containerd
server，里面内容也很丰富，后面单独解释  

## launchService
{{< highlight go "linenos=inline" >}}
app.Action = func(context *cli.Context) error {
     ...
     // Launch as a Windows Service if necessary
     if err := launchService(server, done); err != nil {
         logrus.Fatal(err)
     }
     serverC <- server
     ...
}
这里的launchService主要供windows使用，对于linux而言，直接返回nil，完了之后，会将创建好的server发送给serverC通道，供信号处理函数使用  

## config.Debug.Address
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
app.Action = func(context *cli.Context) error {
    ...
     if config.Debug.Address != "" {
           var l net.Listener
           if filepath.IsAbs(config.Debug.Address) {
               if l, err = sys.GetLocalListener(config.Debug.Address, config.Debug.UID, config.Debug.GID); err != nil {
                   return errors.Wrapf(err, "failed to get listener for debug endpoint")
               }
           } else {
               if l, err = net.Listen("tcp", config.Debug.Address); err != nil {
                   return errors.Wrapf(err, "failed to get listener for debug endpoint")
               }
           }
           serve(ctx, l, server.ServeDebug)
       }
       ...
}
{{< /highlight >}}
如果开启了config.Debug.Address,将设置debug相关的参数  

## config.Metrics 
设置metrics，GRPC相关的参数
{{< highlight go "linenos=inline" >}}
app.Action = func(context *cli.Context) error {
    ...
       if config.Metrics.Address != "" {
           l, err := net.Listen("tcp", config.Metrics.Address)
           if err != nil {
               return errors.Wrapf(err, "failed to get listener for metrics endpoint")
           }
           serve(ctx, l, server.ServeMetrics)
       }
       // setup the ttrpc endpoint
       tl, err := sys.GetLocalListener(ttrpcAddress, config.GRPC.UID, config.GRPC.GID)
       if err != nil {
           return errors.Wrapf(err, "failed to get listener for main ttrpc endpoint")
       }
       serve(ctx, tl, server.ServeTTRPC)
...
{{< /highlight >}}

## config.GRPC
设置metrics，GRPC相关的参数
{{< highlight go "linenos=inline" >}}
       if config.GRPC.TCPAddress != "" {
           l, err := net.Listen("tcp", config.GRPC.TCPAddress)
           if err != nil {
               return errors.Wrapf(err, "failed to get listener for TCP grpc endpoint")
           }
           serve(ctx, l, server.ServeTCP)
       }
       // setup the main grpc endpoint
       l, err := sys.GetLocalListener(address, config.GRPC.UID, config.GRPC.GID)
       if err != nil {
           return errors.Wrapf(err, "failed to get listener for main endpoint")
       }
       serve(ctx, l, server.ServeGRPC)
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
app.Action = func(context *cli.Context) error {
    ...
    log.G(ctx).Infof("containerd successfully booted in %fs", time.Since(start).Seconds())
    <-done
    return nil
}
{{< /highlight >}}
到此，整个app.Action的定义也就结束了。<-done会阻塞在done通道，如果有值到来，才表示可以结束进程了  

# 辅助函数
下面在看前面几个用到的辅助函数   

## serve
serve函数会启动一个协程，执行serveFunc
{{< highlight go "linenos=inline" >}}
func serve(ctx gocontext.Context, l net.Listener, serveFunc func(net.Listener) error) {
    path := l.Addr().String()
    log.G(ctx).WithField("address", path).Info("serving...")
    go func() {
        defer l.Close()
        if err := serveFunc(l); err != nil {
            log.G(ctx).WithError(err).WithField("address", path).Fatal("serve failure")
        }
    }()
}
{{< /highlight >}}

## sys.GetLocalListener
GetLocalListener函数先是确保path目录创建和设置正确的属主，在目录下创建和监听unix
socket，并返回相关的listener
{{< highlight go "linenos=inline" >}}
"github.com/containerd/containerd/sys"
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/sys/socket_unix.go
// GetLocalListener returns a listener out of a unix socket.
func GetLocalListener(path string, uid, gid int) (net.Listener, error) {
    // Ensure parent directory is created
    if err := mkdirAs(filepath.Dir(path), uid, gid); err != nil {
        return nil, err
    }

    l, err := CreateUnixSocket(path)
    if err != nil {
        return l, err
    }

    if err := os.Chmod(path, 0660); err != nil {
        l.Close()
        return nil, err
    }

    if err := os.Chown(path, uid, gid); err != nil {
        l.Close()
        return nil, err
    }

    return l, nil
}
{{< /highlight >}}

## CreateUnixSocket
CreateUnixSocket用于创建一个unix socket，比如设置ttrpc endpoint的时候，path的值是<code>/run/containerd/containerd.sock.ttrpc</code>，unix.Unlink(path)其实会将path文件删掉，但是net.Listen("unix", path)又会重新创建。  

那是不是可以不用Unlink呢？答案是否定的，这里如果去掉unix.Unlink(path)，若上一次containerd的运行导致path文件有残留，那net.Listen("unix", path)会返回错误，并示：listen unix /run/containerd/containerd.sock.ttrpc: bind: address already in use。当然，如果是path文件本身不存在，Unlink会返回<code>no such file or directory</code>，所以后面需要补上!os.IsNotExist(err)来再次确认下错误，如果是不存在导致，那就继续咯。 
{{< highlight go "linenos=inline" >}}
// CreateUnixSocket creates a unix socket and returns the listener
func CreateUnixSocket(path string) (net.Listener, error) {
    // BSDs have a 104 limit
    if len(path) > 104 {
        return nil, errors.Errorf("%q: unix socket path too long (> 104)", path)
    }
    if err := os.MkdirAll(filepath.Dir(path), 0660); err != nil {
        return nil, err
    }
    if err := unix.Unlink(path); err != nil && !os.IsNotExist(err) {
        return nil, err
    }
    return net.Listen("unix", path)
}
{{< /highlight >}}

## unix.Unlink
正如上面看到的，unix.Unlink可用于残留文件删除，实际上，Unlink()函数执行的是引用删除，每一次调用会删除指定路径文件的一次引用，将计数减一，并不一定会真正删除文件，当检查文件系统中此文件的连接数为1，并且在此时没有任何进程打开该文件，执行Unlink将文件引用计数减到0，然后真正删除文件。在有进程打开此文件的情况下，则暂时不会删除，直到所有打开该文件的进程都结束时文件就会被删除。
{{< highlight go "linenos=inline" >}}
import "golang.org/x/sys/unix"
func Unlink(path string) error
{{< /highlight >}}

## net.Listen
返回在一个本地网络地址laddr上监听的Listener。网络类型参数network必须是面向流的网络："tcp"、"tcp4"、"tcp6"、"unix"或"unixpacket"
{{< highlight go "linenos=inline" >}}
func Listen(network, address string) (Listener, error)
{{< /highlight >}}

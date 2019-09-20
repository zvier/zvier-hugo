---
title: "containerd-containerd-cmd/-containerd-command-main"
date: 2019-09-08T16:28:28+08:00
draft: true
categories: [""]
tags: ["containerd"]
---
# 简述
containerd App的主体逻辑
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/containerd/containerd/cmd/containerd/command/main.go
{{< /highlight >}}

# 常量声明
{{< highlight go "linenos=inline" >}}
const usage = `
                    __        _                     __
  _________  ____  / /_____ _(_)___  ___  _________/ /
 / ___/ __ \/ __ \/ __/ __ ` + "`" + `/ / __ \/ _ \/ ___/ __  /
/ /__/ /_/ / / / / /_/ /_/ / / / / /  __/ /  / /_/ /
\___/\____/_/ /_/\__/\__,_/_/_/ /_/\___/_/   \__,_/

high performance container runtime
`
{{< /highlight >}}

# init
init主要是初始化模块的日志系统
{{< highlight go "linenos=inline" >}}
import (
    "github.com/containerd/containerd/log"
    "google.golang.org/grpc/grpclog"
    "github.com/sirupsen/logrus"
    "github.com/urfave/cli"
)
{{< /highlight >}}

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

# App
创建并初始化containerd对应的App
{{< highlight go "linenos=inline" >}}
import (
    "github.com/urfave/cli"
)
{{< /highlight >}}

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
		var (
			start   = time.Now()
			signals = make(chan os.Signal, 2048)
			serverC = make(chan *server.Server, 1)
			ctx     = gocontext.Background()
			config  = defaultConfig()
		)

		if err := srvconfig.LoadConfig(context.GlobalString("config"), config); err != nil && !os.IsNotExist(err) {
			return err
		}

		// Apply flags to the config
		if err := applyFlags(context, config); err != nil {
			return err
		}

		// Make sure top-level directories are created early.
		if err := server.CreateTopLevelDirectories(config); err != nil {
			return err
		}

		// Stop if we are registering or unregistering against Windows SCM.
		stop, err := registerUnregisterService(config.Root)
		if err != nil {
			logrus.Fatal(err)
		}
		if stop {
			return nil
		}

		done := handleSignals(ctx, signals, serverC)
		// start the signal handler as soon as we can to make sure that
		// we don't miss any signals during boot
		signal.Notify(signals, handledSignals...)

		// cleanup temp mounts
		if err := mount.SetTempMountLocation(filepath.Join(config.Root, "tmpmounts")); err != nil {
			return errors.Wrap(err, "creating temp mount location")
		}
		// unmount all temp mounts on boot for the server
		warnings, err := mount.CleanupTempMounts(0)
		if err != nil {
			log.G(ctx).WithError(err).Error("unmounting temp mounts")
		}
		for _, w := range warnings {
			log.G(ctx).WithError(w).Warn("cleanup temp mount")
		}

		if config.GRPC.Address == "" {
			return errors.Wrap(errdefs.ErrInvalidArgument, "grpc address cannot be empty")
		}
		if config.TTRPC.Address == "" {
			// If TTRPC was not explicitly configured, use defaults based on GRPC.
			config.TTRPC.Address = fmt.Sprintf("%s.ttrpc", config.GRPC.Address)
			config.TTRPC.UID = config.GRPC.UID
			config.TTRPC.GID = config.GRPC.GID
		}
		log.G(ctx).WithFields(logrus.Fields{
			"version":  version.Version,
			"revision": version.Revision,
		}).Info("starting containerd")

		server, err := server.New(ctx, config)
		if err != nil {
			return err
		}

		// Launch as a Windows Service if necessary
		if err := launchService(server, done); err != nil {
			logrus.Fatal(err)
		}

		serverC <- server

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
		if config.Metrics.Address != "" {
			l, err := net.Listen("tcp", config.Metrics.Address)
			if err != nil {
				return errors.Wrapf(err, "failed to get listener for metrics endpoint")
			}
			serve(ctx, l, server.ServeMetrics)
		}
		// setup the ttrpc endpoint
		tl, err := sys.GetLocalListener(config.TTRPC.Address, config.TTRPC.UID, config.TTRPC.GID)
		if err != nil {
			return errors.Wrapf(err, "failed to get listener for main ttrpc endpoint")
		}
		serve(ctx, tl, server.ServeTTRPC)

		if config.GRPC.TCPAddress != "" {
			l, err := net.Listen("tcp", config.GRPC.TCPAddress)
			if err != nil {
				return errors.Wrapf(err, "failed to get listener for TCP grpc endpoint")
			}
			serve(ctx, l, server.ServeTCP)
		}
		// setup the main grpc endpoint
		l, err := sys.GetLocalListener(config.GRPC.Address, config.GRPC.UID, config.GRPC.GID)
		if err != nil {
			return errors.Wrapf(err, "failed to get listener for main endpoint")
		}
		serve(ctx, l, server.ServeGRPC)

		log.G(ctx).Infof("containerd successfully booted in %fs", time.Since(start).Seconds())
		<-done
		return nil
	}
	return app
}
{{< /highlight >}}

## 引用

1. service相关的命令行选项: [serviceFlags](http://www.zvier.top/post/containerd-containerd-cmd-containerd-command-service_unsupported/#serviceflags)
2. [configCommand]()
3. [publishCommand]()
4. [ociHook]()
5. [defaultConfig]()
6. [srvconfig.LoadConfig]()
7. [applyFlags]()
8. [server.CreateTopLevelDirectories]()
9. [registerUnregisterService]()
10. [handleSignals]()
11. [handledSignals]()
12. [mount.SetTempMountLocation]()
13. [mount.CleanupTempMounts]()
14. [server.New]()
15. [launchService]()
16. [sys.GetLocalListener]()
17. [serve]()
18. [sys.GetLocalListener]()
19. [server.ServeGRPC]()

# serve
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

# applyFlags
{{< highlight go "linenos=inline" >}}
func applyFlags(context *cli.Context, config *srvconfig.Config) error {
	// the order for config vs flag values is that flags will always override
	// the config values if they are set
	if err := setLevel(context, config); err != nil {
		return err
	}
	for _, v := range []struct {
		name string
		d    *string
	}{
		{
			name: "root",
			d:    &config.Root,
		},
		{
			name: "state",
			d:    &config.State,
		},
		{
			name: "address",
			d:    &config.GRPC.Address,
		},
	} {
		if s := context.GlobalString(v.name); s != "" {
			*v.d = s
		}
	}

	applyPlatformFlags(context)

	return nil
}
{{< /highlight >}}

# setLevel
{{< highlight go "linenos=inline" >}}
func setLevel(context *cli.Context, config *srvconfig.Config) error {
	l := context.GlobalString("log-level")
	if l == "" {
		l = config.Debug.Level
	}
	if l != "" {
		lvl, err := log.ParseLevel(l)
		if err != nil {
			return err
		}
		logrus.SetLevel(lvl)
	}
	return nil
}
{{< /highlight >}}

# dumpStacks
{{< highlight go "linenos=inline" >}}
func dumpStacks(writeToFile bool) {
	var (
		buf       []byte
		stackSize int
	)
	bufferLen := 16384
	for stackSize == len(buf) {
		buf = make([]byte, bufferLen)
		stackSize = runtime.Stack(buf, true)
		bufferLen *= 2
	}
	buf = buf[:stackSize]
	logrus.Infof("=== BEGIN goroutine stack dump ===\n%s\n=== END goroutine stack dump ===", buf)

	if writeToFile {
		// Also write to file to aid gathering diagnostics
		name := filepath.Join(os.TempDir(), fmt.Sprintf("containerd.%d.stacks.log", os.Getpid()))
		f, err := os.Create(name)
		if err != nil {
			return
		}
		defer f.Close()
		f.WriteString(string(buf))
		logrus.Infof("goroutine stack dump written to %s", name)
	}
}
{{< /highlight >}}

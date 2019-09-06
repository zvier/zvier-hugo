---
title: "Containerd Shim 3"
date: 2019-06-23T17:54:13+08:00
draft: true
categories: [""]
tags: [""]
---

# 主函数——main

{{< highlight go "linenos=inline" >}}
 func main() {
     debug.SetGCPercent(40)
     go func() {
         for range time.Tick(30 * time.Second) {
             debug.FreeOSMemory()
         }
     }()

     if debugFlag {
         logrus.SetLevel(logrus.DebugLevel)
     }

     if os.Getenv("GOMAXPROCS") == "" {
         // If GOMAXPROCS hasn't been set, we default to a value of 2 to reduce
         // the number of Go stacks present in the shim.
         runtime.GOMAXPROCS(2)
     }

     stdout, stderr, err := openStdioKeepAlivePipes(workdirFlag)
     if err != nil {
         fmt.Fprintf(os.Stderr, "containerd-shim: %s\n", err)
         os.Exit(1)
     }
     defer func() {
         stdout.Close()
         stderr.Close()
     }()

     // redirect the following output into fifo to make sure that containerd
     // still can read the log after restart
     logrus.SetOutput(stdout)

     if err := executeShim(); err != nil {
         fmt.Fprintf(os.Stderr, "containerd-shim: %s\n", err)
        os.Exit(1)
     }
 }
{{< /highlight >}}


# openStdioKeepAlivePipes

{{< highlight go "linenos=inline" >}}
// If containerd server process dies, we need the shim to keep stdout/err reader
// FDs so that Linux does not SIGPIPE the shim process if it tries to use its end of
// these pipes.
func openStdioKeepAlivePipes(dir string) (io.ReadWriteCloser, io.ReadWriteCloser, error) {
    background := context.Background()
    keepStdoutAlive, err := shimlog.OpenShimStdoutLog(background, dir)
    if err != nil {
        return nil, nil, err
    }
    keepStderrAlive, err := shimlog.OpenShimStderrLog(background, dir)
    if err != nil {
        return nil, nil, err
    }
    return keepStdoutAlive, keepStderrAlive, nil
}
{{< /highlight >}}
openStdioKeepAlivePipes在[cloudfoundry/guardian](https://github.com/cloudfoundry/guardian/blob/master/cmd/dadoo/main_linux.go)也有类似的实现，一旦containerd
server进程挂了，containerd-shim可以确保stdout/err文件描述符不会被关闭。

关于SIGPIPE的介绍参考[SIGPIPE信号详解]()

# executeShim
{{< highlight go "linenos=inline" >}}
func executeShim() error {
	// start handling signals as soon as possible so that things are properly reaped
	// or if runtime exits before we hit the handler
	signals, err := setupSignals()
	if err != nil {
		return err
	}
	dump := make(chan os.Signal, 32)
	signal.Notify(dump, syscall.SIGUSR1)

	path, err := os.Getwd()
	if err != nil {
		return err
	}
	server, err := newServer()
	if err != nil {
		return errors.Wrap(err, "failed creating server")
	}
	sv, err := shim.NewService(
		shim.Config{
			Path:          path,
			Namespace:     namespaceFlag,
			WorkDir:       workdirFlag,
			Criu:          criuFlag,
			SystemdCgroup: systemdCgroupFlag,
			RuntimeRoot:   runtimeRootFlag,
		},
		&remoteEventsPublisher{address: addressFlag},
	)
	if err != nil {
		return err
	}
	logrus.Debug("registering ttrpc server")
	shimapi.RegisterShimService(server, sv)

	socket := socketFlag
	if err := serve(context.Background(), server, socket); err != nil {
		return err
	}
	logger := logrus.WithFields(logrus.Fields{
		"pid":       os.Getpid(),
		"path":      path,
		"namespace": namespaceFlag,
	})
	go func() {
		for range dump {
			dumpStacks(logger)
		}
	}()
	return handleSignals(logger, signals, server, sv)
}
{{< /highlight >}}


# serve

{{< highlight go "linenos=inline" >}}
// serve serves the ttrpc API over a unix socket at the provided path
// this function does not block
func serve(ctx context.Context, server *ttrpc.Server, path string) error {
	var (
		l   net.Listener
		err error
	)
	if path == "" {
		f := os.NewFile(3, "socket")
		l, err = net.FileListener(f)
		f.Close()
		path = "[inherited from parent]"
	} else {
		if len(path) > 106 {
			return errors.Errorf("%q: unix socket path too long (> 106)", path)
		}
		l, err = net.Listen("unix", "\x00"+path)
	}
	if err != nil {
		return err
	}
	logrus.WithField("socket", path).Debug("serving api on unix socket")
	go func() {
		defer l.Close()
		if err := server.Serve(ctx, l); err != nil &&
			!strings.Contains(err.Error(), "use of closed network connection") {
			logrus.WithError(err).Fatal("containerd-shim: ttrpc server failure")
		}
	}()
	return nil
}
{{< /highlight >}}


# handleSignals
{{< highlight go "linenos=inline" >}}
func handleSignals(logger *logrus.Entry, signals chan os.Signal, server *ttrpc.Server, sv *shim.Service) error {
	var (
		termOnce sync.Once
		done     = make(chan struct{})
	)

	for {
		select {
		case <-done:
			return nil
		case s := <-signals:
			switch s {
			case unix.SIGCHLD:
				if err := shim.Reap(); err != nil {
					logger.WithError(err).Error("reap exit status")
				}
			case unix.SIGTERM, unix.SIGINT:
				go termOnce.Do(func() {
					ctx := context.TODO()
					if err := server.Shutdown(ctx); err != nil {
						logger.WithError(err).Error("failed to shutdown server")
					}
					// Ensure our child is dead if any
					sv.Kill(ctx, &shimapi.KillRequest{
						Signal: uint32(syscall.SIGKILL),
						All:    true,
					})
					sv.Delete(context.Background(), &ptypes.Empty{})
					close(done)
				})
			case unix.SIGPIPE:
			}
		}
	}
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
func dumpStacks(logger *logrus.Entry) {
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
	logger.Infof("=== BEGIN goroutine stack dump ===\n%s\n=== END goroutine stack dump ===", buf)
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
type remoteEventsPublisher struct {
	address string
}

func (l *remoteEventsPublisher) Publish(ctx context.Context, topic string, event events.Event) error {
	ns, _ := namespaces.Namespace(ctx)
	encoded, err := typeurl.MarshalAny(event)
	if err != nil {
		return err
	}
	data, err := encoded.Marshal()
	if err != nil {
		return err
	}
	cmd := exec.CommandContext(ctx, containerdBinaryFlag, "--address", l.address, "publish", "--topic", topic, "--namespace", ns)
	cmd.Stdin = bytes.NewReader(data)
	b := bufPool.Get().(*bytes.Buffer)
	defer bufPool.Put(b)
	cmd.Stdout = b
	cmd.Stderr = b
	c, err := shim.Default.Start(cmd)
	if err != nil {
		return err
	}
	status, err := shim.Default.Wait(cmd, c)
	if err != nil {
		return errors.Wrapf(err, "failed to publish event: %s", b.String())
	}
	if status != 0 {
		return errors.Errorf("failed to publish event: %s", b.String())
	}
	return nil
}
{{< /highlight >}}
remoteEventsPublisher主要是与containerd进行grpc通信，address即containerd server地址，默认为/run/containerd/containerd.sock，关于namespaces可以参考，typeurl参考，encoded参考，exec.CommandContext参考，

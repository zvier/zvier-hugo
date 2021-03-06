---
title: "github.com opencontainers runc main"
date: 2020-03-21T14:55:00+08:00
draft: true
categories: ["技术"]
tags: ["runc"]
---
桃李春风一杯酒，江湖夜雨十年灯。——黄庭坚《寄黄几复》
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/opencontainers/runc/main.go
{{< /highlight >}}

# 变量
{{< highlight go "linenos=inline" >}}
// version will be populated by the Makefile, read from
// VERSION file of the source code.
var version = ""

// gitCommit will be the hash that the binary was built from
// and will be populated by the Makefile
var gitCommit = ""
{{< /highlight >}}

# 常量
{{< highlight go "linenos=inline" >}}
const (
	specConfig = "config.json"
	usage      = `Open Container Initiative runtime

runc is a command line client for running applications packaged according to
the Open Container Initiative (OCI) format and is a compliant implementation of the
Open Container Initiative specification.

runc integrates well with existing process supervisors to provide a production
container runtime environment for applications. It can be used with your
existing process monitoring tools and the container will be spawned as a
direct child of the process supervisor.

Containers are configured using bundles. A bundle for a container is a directory
that includes a specification file named "` + specConfig + `" and a root filesystem.
The root filesystem contains the contents of the container.

To start a new instance of a container:

    # runc run [ -b bundle ] <container-id>

Where "<container-id>" is your name for the instance of the container that you
are starting. The name you provide for the container instance must be unique on
your host. Providing the bundle directory using "-b" is optional. The default
value for "bundle" is the current directory.`
)
{{< /highlight >}}

# main
{{< highlight go "linenos=inline" >}}
func main() {
	app := cli.NewApp()
	app.Name = "runc"
	app.Usage = usage

	var v []string
	if version != "" {
		v = append(v, version)
	}
	if gitCommit != "" {
		v = append(v, fmt.Sprintf("commit: %s", gitCommit))
	}
	v = append(v, fmt.Sprintf("spec: %s", specs.Version))
	app.Version = strings.Join(v, "\n")

	root := "/run/runc"
	if shouldHonorXDGRuntimeDir() {
		if runtimeDir := os.Getenv("XDG_RUNTIME_DIR"); runtimeDir != "" {
			root = runtimeDir + "/runc"
			// According to the XDG specification, we need to set anything in
			// XDG_RUNTIME_DIR to have a sticky bit if we don't want it to get
			// auto-pruned.
			if err := os.MkdirAll(root, 0700); err != nil {
				fatal(err)
			}
			if err := os.Chmod(root, 0700|os.ModeSticky); err != nil {
				fatal(err)
			}
		}
	}

	app.Flags = []cli.Flag{
		cli.BoolFlag{
			Name:  "debug",
			Usage: "enable debug output for logging",
		},
		cli.StringFlag{
			Name:  "log",
			Value: "",
			Usage: "set the log file path where internal debug information is written",
		},
		cli.StringFlag{
			Name:  "log-format",
			Value: "text",
			Usage: "set the format used by logs ('text' (default), or 'json')",
		},
		cli.StringFlag{
			Name:  "root",
			Value: root,
			Usage: "root directory for storage of container state (this should be located in tmpfs)",
		},
		cli.StringFlag{
			Name:  "criu",
			Value: "criu",
			Usage: "path to the criu binary used for checkpoint and restore",
		},
		cli.BoolFlag{
			Name:  "systemd-cgroup",
			Usage: "enable systemd cgroup support, expects cgroupsPath to be of form \"slice:prefix:name\" for e.g. \"system.slice:runc:434234\"",
		},
		cli.StringFlag{
			Name:  "rootless",
			Value: "auto",
			Usage: "ignore cgroup permission errors ('true', 'false', or 'auto')",
		},
	}
	app.Commands = []cli.Command{
		checkpointCommand,
		createCommand,
		deleteCommand,
		eventsCommand,
		execCommand,
		initCommand,
		killCommand,
		listCommand,
		pauseCommand,
		psCommand,
		restoreCommand,
		resumeCommand,
		runCommand,
		specCommand,
		startCommand,
		stateCommand,
		updateCommand,
	}
	app.Before = func(context *cli.Context) error {
		return logs.ConfigureLogging(createLogConfig(context))
	}

	// If the command returns an error, cli takes upon itself to print
	// the error on cli.ErrWriter and exit.
	// Use our own writer here to ensure the log gets sent to the right location.
	cli.ErrWriter = &FatalWriter{cli.ErrWriter}
	if err := app.Run(os.Args); err != nil {
		fatal(err)
	}
}
{{< /highlight >}}

# FatalWriter
{{< highlight go "linenos=inline" >}}
type FatalWriter struct {
	cliErrWriter io.Writer
}
{{< /highlight >}}

# FatalWriter.Write
{{< highlight go "linenos=inline" >}}
func (f *FatalWriter) Write(p []byte) (n int, err error) {
	logrus.Error(string(p))
	return f.cliErrWriter.Write(p)
}
{{< /highlight >}}

# createLogConfig
{{< highlight go "linenos=inline" >}}
func createLogConfig(context *cli.Context) logs.Config {
	logFilePath := context.GlobalString("log")
	logPipeFd := ""
	if logFilePath == "" {
		logPipeFd = "2"
	}
	config := logs.Config{
		LogPipeFd:   logPipeFd,
		LogLevel:    logrus.InfoLevel,
		LogFilePath: logFilePath,
		LogFormat:   context.GlobalString("log-format"),
	}
	if context.GlobalBool("debug") {
		config.LogLevel = logrus.DebugLevel
	}

	return config
}
{{< /highlight >}}


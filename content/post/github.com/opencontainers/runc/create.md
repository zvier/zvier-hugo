---
title: "github.com opencontainers runc create"
date: 2020-03-21T18:06:13+08:00
draft: true
categories: ["技术"]
tags: ["runc"]
---
雨中黄叶树，灯下白头人。——司空曙《喜外弟卢纶见宿》
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/opencontainers/runc/create.go
{{< /highlight >}}

# createCommand
create子命令，用于创建一个容器，比如runc create busybox将根据容器文件系统和配置文件创建一个名为busybox的容器，其中容器文件系统和配置文件构成一个bundle。
{{< highlight go "linenos=inline" >}}
var createCommand = cli.Command{
	Name:  "create",
	Usage: "create a container",
	ArgsUsage: `<container-id>

Where "<container-id>" is your name for the instance of the container that you
are starting. The name you provide for the container instance must be unique on
your host.`,
	Description: `The create command creates an instance of a container for a bundle. The bundle
is a directory with a specification file named "` + specConfig + `" and a root
filesystem.

The specification file includes an args parameter. The args parameter is used
to specify command(s) that get run when the container is started. To change the
command(s) that get executed on start, edit the args parameter of the spec. See
"runc spec --help" for more explanation.`,
	Flags: []cli.Flag{
		cli.StringFlag{
			Name:  "bundle, b",
			Value: "",
			Usage: `path to the root of the bundle directory, defaults to the current directory`,
		},
		cli.StringFlag{
			Name:  "console-socket",
			Value: "",
			Usage: "path to an AF_UNIX socket which will receive a file descriptor referencing the master end of the console's pseudoterminal",
		},
		cli.StringFlag{
			Name:  "pid-file",
			Value: "",
			Usage: "specify the file to write the process id to",
		},
		cli.BoolFlag{
			Name:  "no-pivot",
			Usage: "do not use pivot root to jail process inside rootfs.  This should be used whenever the rootfs is on top of a ramdisk",
		},
		cli.BoolFlag{
			Name:  "no-new-keyring",
			Usage: "do not create a new session keyring for the container.  This will cause the container to inherit the calling processes session key",
		},
		cli.IntFlag{
			Name:  "preserve-fds",
			Usage: "Pass N additional file descriptors to the container (stdio + $LISTEN_FDS + N in total)",
		},
	},
	Action: func(context *cli.Context) error {
		if err := checkArgs(context, 1, exactArgs); err != nil {
			return err
		}
		if err := revisePidFile(context); err != nil {
			return err
		}
		spec, err := setupSpec(context)
		if err != nil {
			return err
		}
		status, err := startContainer(context, spec, CT_ACT_CREATE, nil)
		if err != nil {
			return err
		}
		// exit with the container's exit status so any external supervisor is
		// notified of the exit with the correct exit status.
		os.Exit(status)
		return nil
	},
}
{{< /highlight >}}

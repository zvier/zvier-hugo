---
title: "docker build (1)"
date: 2020-02-09T13:09:26+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
# 简述
<!--more-->
# main
{{< highlight go "linenos=inline" >}}
// github.com/docker/cli/cmd/docker/docker.go
func main() {
    dockerCli, err := command.NewDockerCli()
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
    logrus.SetOutput(dockerCli.Err())

    if err := runDocker(dockerCli); err != nil {
        if sterr, ok := err.(cli.StatusError); ok {
            if sterr.Status != "" {
                fmt.Fprintln(dockerCli.Err(), sterr.Status)
            }
            // StatusError should only be used for errors, and all errors should
            // have a non-zero exit status, so never exit with 0
            if sterr.StatusCode == 0 {
                os.Exit(1)
            }
            os.Exit(sterr.StatusCode)
        }
        fmt.Fprintln(dockerCli.Err(), err)
        os.Exit(1)
    }
}
{{< /highlight >}}

## command.NewDockerCli
{{< highlight go "linenos=inline" >}}
// github.com/docker/cli/cli/command/cli.go
// NewDockerCli returns a DockerCli instance with all operators applied on it.
// It applies by default the standard streams, and the content trust from
// environment.
func NewDockerCli(ops ...DockerCliOption) (*DockerCli, error) {
    cli := &DockerCli{}
    defaultOps := []DockerCliOption{
        WithContentTrustFromEnv(),
    }
    cli.contextStoreConfig = DefaultContextStoreConfig()
    ops = append(defaultOps, ops...)
    if err := cli.Apply(ops...); err != nil {
        return nil, err
    }
    if cli.out == nil || cli.in == nil || cli.err == nil {
        stdin, stdout, stderr := term.StdStreams()
        if cli.in == nil {
            cli.in = streams.NewIn(stdin)
        }
        if cli.out == nil {
            cli.out = streams.NewOut(stdout)
        }
        if cli.err == nil {
            cli.err = stderr
        }
    }
    return cli, nil
}
{{< /highlight >}}

# runDocker
{{< highlight go "linenos=inline" >}}
func runDocker(dockerCli *command.DockerCli) error {
    tcmd := newDockerCommand(dockerCli)

    cmd, args, err := tcmd.HandleGlobalFlags()
    if err != nil {
        return err
    }

    if err := tcmd.Initialize(); err != nil {
        return err
    }

    args, os.Args, err = processAliases(dockerCli, cmd, args, os.Args)
    if err != nil {
        return err
    }

    if len(args) > 0 {
        if _, _, err := cmd.Find(args); err != nil {
            err := tryPluginRun(dockerCli, cmd, args[0])
            if !pluginmanager.IsNotFound(err) {
                return err
            }
            // For plugin not found we fall through to
            // cmd.Execute() which deals with reporting
            // "command not found" in a consistent way.
        }
    }

    // We've parsed global args already, so reset args to those
    // which remain.
    cmd.SetArgs(args)
    return cmd.Execute()
}
{{< /highlight >}}

# newDockerCommand
{{< highlight go "linenos=inline" >}}
func newDockerCommand(dockerCli *command.DockerCli) *cli.TopLevelCommand {
    var (
        opts    *cliflags.ClientOptions
        flags   *pflag.FlagSet
        helpCmd *cobra.Command
    )

    cmd := &cobra.Command{
        Use:              "docker [OPTIONS] COMMAND [ARG...]",
        Short:            "A self-sufficient runtime for containers",
        SilenceUsage:     true,
        SilenceErrors:    true,
        TraverseChildren: true,
        RunE: func(cmd *cobra.Command, args []string) error {
            if len(args) == 0 {
                return command.ShowHelp(dockerCli.Err())(cmd, args)
            }
            return fmt.Errorf("docker: '%s' is not a docker command.\nSee 'docker --help'", args[0])

        },
        PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
            return isSupported(cmd, dockerCli)
        },
        Version:               fmt.Sprintf("%s, build %s", version.Version, version.GitCommit),
        DisableFlagsInUseLine: true,
    }
    opts, flags, helpCmd = cli.SetupRootCommand(cmd)
    flags.BoolP("version", "v", false, "Print version information and quit")

    setFlagErrorFunc(dockerCli, cmd)
    setupHelpCommand(dockerCli, cmd, helpCmd)
    setHelpFunc(dockerCli, cmd)

    cmd.SetOutput(dockerCli.Out())
    commands.AddCommands(cmd, dockerCli)

    cli.DisableFlagsInUseLine(cmd)
    setValidateArgs(dockerCli, cmd)

    // flags must be the top-level command flags, not cmd.Flags()
    return cli.NewTopLevelCommand(cmd, dockerCli, opts, flags)
}
{{< /highlight >}}

# cli.SetupRootCommand
{{< highlight go "linenos=inline" >}}
// /github.com/docker/cli/cli/cobra.go
// setupCommonRootCommand contains the setup common to
// SetupRootCommand and SetupPluginRootCommand.
func setupCommonRootCommand(rootCmd *cobra.Command) (*cliflags.ClientOptions, *pflag.FlagSet, *cobra.Command) {
    opts := cliflags.NewClientOptions()
    flags := rootCmd.Flags()

    flags.StringVar(&opts.ConfigDir, "config", cliconfig.Dir(), "Location of client config files")
    opts.Common.InstallFlags(flags)

    cobra.AddTemplateFunc("add", func(a, b int) int { return a + b })
    cobra.AddTemplateFunc("hasSubCommands", hasSubCommands)
    cobra.AddTemplateFunc("hasManagementSubCommands", hasManagementSubCommands)
    cobra.AddTemplateFunc("hasInvalidPlugins", hasInvalidPlugins)
    cobra.AddTemplateFunc("operationSubCommands", operationSubCommands)
    cobra.AddTemplateFunc("managementSubCommands", managementSubCommands)
    cobra.AddTemplateFunc("invalidPlugins", invalidPlugins)
    cobra.AddTemplateFunc("wrappedFlagUsages", wrappedFlagUsages)
    cobra.AddTemplateFunc("vendorAndVersion", vendorAndVersion)
    cobra.AddTemplateFunc("invalidPluginReason", invalidPluginReason)
    cobra.AddTemplateFunc("isPlugin", isPlugin)
    cobra.AddTemplateFunc("decoratedName", decoratedName)

    rootCmd.SetUsageTemplate(usageTemplate)
    rootCmd.SetHelpTemplate(helpTemplate)
    rootCmd.SetFlagErrorFunc(FlagErrorFunc)
    rootCmd.SetHelpCommand(helpCommand)

    rootCmd.PersistentFlags().BoolP("help", "h", false, "Print usage")
    rootCmd.PersistentFlags().MarkShorthandDeprecated("help", "please use --help")
    rootCmd.PersistentFlags().Lookup("help").Hidden = true

    return opts, flags, helpCommand
}
{{< /highlight >}}

# commands.AddComands
{{< highlight go "linenos=inline" >}}
// github.com/docker/cli/cli/command/commands/commands.go
// AddCommands adds all the commands from cli/command to the root command
func AddCommands(cmd *cobra.Command, dockerCli command.Cli) {
    cmd.AddCommand(
        // checkpoint
        checkpoint.NewCheckpointCommand(dockerCli),

        // config
        config.NewConfigCommand(dockerCli),

        // container
        container.NewContainerCommand(dockerCli),
        container.NewRunCommand(dockerCli),

        // image
        image.NewImageCommand(dockerCli),
        image.NewBuildCommand(dockerCli),

        // builder
        builder.NewBuilderCommand(dockerCli),

        // manifest
        manifest.NewManifestCommand(dockerCli),

        // network
        network.NewNetworkCommand(dockerCli),

        // node
        node.NewNodeCommand(dockerCli),

        // plugin
        plugin.NewPluginCommand(dockerCli),

        // registry
        registry.NewLoginCommand(dockerCli),
        registry.NewLogoutCommand(dockerCli),
        registry.NewSearchCommand(dockerCli),

        // trust
        trust.NewTrustCommand(dockerCli),

        // volume
        volume.NewVolumeCommand(dockerCli),

        // context
        context.NewContextCommand(dockerCli),

{{< /highlight >}}

# image.NewBuildCommand
{{< highlight go "linenos=inline" >}}
// github.com/docker/cli/cli/command/image/build.go
// NewBuildCommand creates a new `docker build` command
func NewBuildCommand(dockerCli command.Cli) *cobra.Command {
    options := newBuildOptions()

    cmd := &cobra.Command{
        Use:   "build [OPTIONS] PATH | URL | -",
        Short: "Build an image from a Dockerfile",
        Args:  cli.ExactArgs(1),
        RunE: func(cmd *cobra.Command, args []string) error {
            options.context = args[0]
            return runBuild(dockerCli, options)
        },
    }

    flags := cmd.Flags()

    flags.VarP(&options.tags, "tag", "t", "Name and optionally a tag in the 'name:tag' format")
    flags.Var(&options.buildArgs, "build-arg", "Set build-time variables")
    flags.Var(options.ulimits, "ulimit", "Ulimit options")
    flags.SetAnnotation("ulimit", "no-buildkit", nil)
    flags.StringVarP(&options.dockerfileName, "file", "f", "", "Name of the Dockerfile (Default is 'PATH/Dockerfile')")
    flags.VarP(&options.memory, "memory", "m", "Memory limit")
    flags.SetAnnotation("memory", "no-buildkit", nil)
    ...
    flags.SetAnnotation("ssh", "version", []string{"1.39"})
    flags.SetAnnotation("ssh", "buildkit", nil)
 
    flags.StringArrayVarP(&options.outputs, "output", "o", []string{}, "Output destination (format: type=local,dest=path)")
    flags.SetAnnotation("output", "version", []string{"1.40"})
    flags.SetAnnotation("output", "buildkit", nil)
 
    return cmd
 }

{{< /highlight >}}

# newBuildOptions
{{< highlight go "linenos=inline" >}}
// github.com/docker/cli/cli/command/image/build.go
func newBuildOptions() buildOptions {
    ulimits := make(map[string]*units.Ulimit)
    return buildOptions{
        tags:       opts.NewListOpts(validateTag),
        buildArgs:  opts.NewListOpts(opts.ValidateEnv),
        ulimits:    opts.NewUlimitOpt(&ulimits),
        labels:     opts.NewListOpts(opts.ValidateLabel),
        extraHosts: opts.NewListOpts(opts.ValidateExtraHost),
    }
}
{{< /highlight >}}

# buildOptions
{{< highlight go "linenos=inline" >}}
// github.com/docker/cli/cli/command/image/build.go
 type buildOptions struct {
     context        string
     dockerfileName string
     tags           opts.ListOpts
     labels         opts.ListOpts
     buildArgs      opts.ListOpts
     extraHosts     opts.ListOpts
     ulimits        *opts.UlimitOpt
     memory         opts.MemBytes
     memorySwap     opts.MemSwapBytes
     shmSize        opts.MemBytes
     cpuShares      int64
     cpuPeriod      int64
     cpuQuota       int64
     cpuSetCpus     string
     cpuSetMems     string
     ....
     forceRm        bool
     pull           bool
     cacheFrom      []string
     compress       bool
     securityOpt    []string
     networkMode    string
     squash         bool
     target         string
     imageIDFile    string
     stream         bool
     platform       string
     untrusted      bool
     secrets        []string
     ssh            []string
     outputs        []string
 }
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

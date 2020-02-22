---
title: "docker build (3)"
date: 2020-02-09T21:37:49+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
远远围墙，隐隐茅堂。飏青旗、流水桥旁。——秦观《行香子·树绕村庄》
<!--more-->
# runBuildBuildKit
{{< highlight go "linenos=inline" >}}
// github.com/docker/cli/cli/command/image/build_buildkit.go
//nolint: gocyclo
func runBuildBuildKit(dockerCli command.Cli, options buildOptions) error {
    ctx := appcontext.Context()

    s, err := trySession(dockerCli, options.context, false)
    if err != nil {
        return err
    }
    if s == nil {
        return errors.Errorf("buildkit not supported by daemon")
    }

    if options.imageIDFile != "" {
        // Avoid leaving a stale file if we eventually fail
        if err := os.Remove(options.imageIDFile); err != nil && !os.IsNotExist(err) {
            return errors.Wrap(err, "removing image ID file")
        }
    }
    var (
        remote           string
        body             io.Reader
        dockerfileName   = options.dockerfileName
        dockerfileReader io.ReadCloser
        dockerfileDir    string
        contextDir       string
    )

    stdoutUsed := false

    switch {
    case options.contextFromStdin():
        if options.dockerfileFromStdin() {
            return errStdinConflict
        }
        rc, isArchive, err := build.DetectArchiveReader(os.Stdin)
        if err != nil {
            return err
        }
        if isArchive {
            body = rc
            remote = uploadRequestRemote
        } else {
            if options.dockerfileName != "" {
                return errDockerfileConflict
            }
            dockerfileReader = rc
            remote = clientSessionRemote
            // TODO: make fssync handle empty contextdir
            contextDir, _ = ioutil.TempDir("", "empty-dir")
            defer os.RemoveAll(contextDir)
        }
    case isLocalDir(options.context):
        contextDir = options.context
        if options.dockerfileFromStdin() {
            dockerfileReader = os.Stdin
        } else if options.dockerfileName != "" {
            dockerfileName = filepath.Base(options.dockerfileName)
            dockerfileDir = filepath.Dir(options.dockerfileName)
        } else {
            dockerfileDir = options.context
        }
        remote = clientSessionRemote
    case urlutil.IsGitURL(options.context):
        remote = options.context
    case urlutil.IsURL(options.context):
        remote = options.context
    default:
        return errors.Errorf("unable to prepare context: path %q not found", options.context)
    }

    if dockerfileReader != nil {
        dockerfileName = build.DefaultDockerfileName
        dockerfileDir, err = build.WriteTempDockerfile(dockerfileReader)
        if err != nil {
            return err
        }
        defer os.RemoveAll(dockerfileDir)
    }

    outputs, err := parseOutputs(options.outputs)
    if err != nil {
        return errors.Wrapf(err, "failed to parse outputs")
    }

    for _, out := range outputs {
        switch out.Type {
        case "local":
            // dest is handled on client side for local exporter
            outDir, ok := out.Attrs["dest"]
            if !ok {
                return errors.Errorf("dest is required for local output")
            }
            delete(out.Attrs, "dest")
            s.Allow(filesync.NewFSSyncTargetDir(outDir))
        case "tar":
            // dest is handled on client side for tar exporter
            outFile, ok := out.Attrs["dest"]
            if !ok {
                return errors.Errorf("dest is required for tar output")
            }
            var w io.WriteCloser
            if outFile == "-" {
                if _, err := console.ConsoleFromFile(os.Stdout); err == nil {
                    return errors.Errorf("refusing to write output to console")
                }
                w = os.Stdout
                stdoutUsed = true
            } else {
                f, err := os.Create(outFile)
                if err != nil {
                    return errors.Wrapf(err, "failed to open %s", outFile)
                }
                w = f
            }
            output := func(map[string]string) (io.WriteCloser, error) { return w, nil }
            s.Allow(filesync.NewFSSyncTarget(output))
        }
    }

    if dockerfileDir != "" {
        s.Allow(filesync.NewFSSyncProvider([]filesync.SyncedDir{
            {
                Name: "context",
                Dir:  contextDir,
                Map:  resetUIDAndGID,
            },
            {
                Name: "dockerfile",
                Dir:  dockerfileDir,
            },
        }))
    }
    s.Allow(authprovider.NewDockerAuthProvider(os.Stderr))
    if len(options.secrets) > 0 {
        sp, err := parseSecretSpecs(options.secrets)
        if err != nil {
            return errors.Wrapf(err, "could not parse secrets: %v", options.secrets)
        }
        s.Allow(sp)
    }
    if len(options.ssh) > 0 {
        sshp, err := parseSSHSpecs(options.ssh)
        if err != nil {
            return errors.Wrapf(err, "could not parse ssh: %v", options.ssh)
        }
        s.Allow(sshp)
    }

    eg, ctx := errgroup.WithContext(ctx)

    dialSession := func(ctx context.Context, proto string, meta map[string][]string) (net.Conn, error) {
        return dockerCli.Client().DialHijack(ctx, "/session", proto, meta)
    }
    eg.Go(func() error {
        return s.Run(context.TODO(), dialSession)
    })
    buildID := stringid.GenerateRandomID()
    if body != nil {
        eg.Go(func() error {
            buildOptions := types.ImageBuildOptions{
                Version: types.BuilderBuildKit,
                BuildID: uploadRequestRemote + ":" + buildID,
            }

            response, err := dockerCli.Client().ImageBuild(context.Background(), body, buildOptions)
            if err != nil {
                return err
            }
            defer response.Body.Close()
            return nil
        })
    }
    if v := os.Getenv("BUILDKIT_PROGRESS"); v != "" && options.progress == "auto" {
        options.progress = v
    }

    if strings.EqualFold(options.platform, "local") {
        options.platform = platforms.DefaultString()
    }

    eg.Go(func() error {
        defer func() { // make sure the Status ends cleanly on build errors
            s.Close()
        }()

        buildOptions := imageBuildOptions(dockerCli, options)
        buildOptions.Version = types.BuilderBuildKit
        buildOptions.Dockerfile = dockerfileName
        // buildOptions.AuthConfigs = authConfigs   // handled by session
        buildOptions.RemoteContext = remote
        buildOptions.SessionID = s.ID()
        buildOptions.BuildID = buildID
        buildOptions.Outputs = outputs
        return doBuild(ctx, eg, dockerCli, stdoutUsed, options, buildOptions)
    })

    return eg.Wait()
}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

# 引用
[appcontext.Context()](http://www.zvier.top/post/appcontext/)

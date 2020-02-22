---
title: "docker build (2)"
date: 2020-02-09T19:36:40+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
山中何事？松花酿酒，春水煎茶。——张可久《人月圆·山中书事》
<!--more-->
# runBuild
{{< highlight go "linenos=inline" >}}
// github.com/docker/cli/cli/command/image/build.go
// nolint: gocyclo
func runBuild(dockerCli command.Cli, options buildOptions) error {
    buildkitEnabled, err := command.BuildKitEnabled(dockerCli.ServerInfo())
    if err != nil {
        return err
    }
    if buildkitEnabled {
        return runBuildBuildKit(dockerCli, options)
    }

    var (
        buildCtx      io.ReadCloser
        dockerfileCtx io.ReadCloser
        contextDir    string
        tempDir       string
        relDockerfile string
        progBuff      io.Writer
        buildBuff     io.Writer
        remote        string
    )

    if options.stream {
        return errors.New("Experimental flag --stream was removed, enable BuildKit instead with DOCKER_BUILDKIT=1")
    }
    if options.dockerfileFromStdin() {
        if options.contextFromStdin() {
            return errStdinConflict
        }
        dockerfileCtx = dockerCli.In()
    }

    specifiedContext := options.context
    progBuff = dockerCli.Out()
    buildBuff = dockerCli.Out()
    if options.quiet {
        progBuff = bytes.NewBuffer(nil)
        buildBuff = bytes.NewBuffer(nil)
    }
    if options.imageIDFile != "" {
        // Avoid leaving a stale file if we eventually fail
        if err := os.Remove(options.imageIDFile); err != nil && !os.IsNotExist(err) {
            return errors.Wrap(err, "Removing image ID file")
        }
    }
    switch {
    case options.contextFromStdin():
        // buildCtx is tar archive. if stdin was dockerfile then it is wrapped
        buildCtx, relDockerfile, err = build.GetContextFromReader(dockerCli.In(), options.dockerfileName)
    case isLocalDir(specifiedContext):
        contextDir, relDockerfile, err = build.GetContextFromLocalDir(specifiedContext, options.dockerfileName)
        if err == nil && strings.HasPrefix(relDockerfile, ".."+string(filepath.Separator)) {
            // Dockerfile is outside of build-context; read the Dockerfile and pass it as dockerfileCtx
            dockerfileCtx, err = os.Open(options.dockerfileName)
            if err != nil {
                return errors.Errorf("unable to open Dockerfile: %v", err)
            }
            defer dockerfileCtx.Close()
        }
    case urlutil.IsGitURL(specifiedContext):
        tempDir, relDockerfile, err = build.GetContextFromGitURL(specifiedContext, options.dockerfileName)
    case urlutil.IsURL(specifiedContext):
        buildCtx, relDockerfile, err = build.GetContextFromURL(progBuff, specifiedContext, options.dockerfileName)
    default:
        return errors.Errorf("unable to prepare context: path %q not found", specifiedContext)
    }

    if err != nil {
        if options.quiet && urlutil.IsURL(specifiedContext) {
            fmt.Fprintln(dockerCli.Err(), progBuff)
        }
        return errors.Errorf("unable to prepare context: %s", err)
    }

    if tempDir != "" {
        defer os.RemoveAll(tempDir)
        contextDir = tempDir
    }
    // read from a directory into tar archive
    if buildCtx == nil {
        excludes, err := build.ReadDockerignore(contextDir)
        if err != nil {
            return err
        }

        if err := build.ValidateContextDirectory(contextDir, excludes); err != nil {
            return errors.Errorf("error checking context: '%s'.", err)
        }

        // And canonicalize dockerfile name to a platform-independent one
        relDockerfile = archive.CanonicalTarNameForPath(relDockerfile)

        excludes = build.TrimBuildFilesFromExcludes(excludes, relDockerfile, options.dockerfileFromStdin())
        buildCtx, err = archive.TarWithOptions(contextDir, &archive.TarOptions{
            ExcludePatterns: excludes,
            ChownOpts:       &idtools.Identity{UID: 0, GID: 0},
        })
        if err != nil {
            return err
        }
    }

    // replace Dockerfile if it was added from stdin or a file outside the build-context, and there is archive context
    if dockerfileCtx != nil && buildCtx != nil {
        buildCtx, relDockerfile, err = build.AddDockerfileToBuildContext(dockerfileCtx, buildCtx)
        if err != nil {
            return err
        }
    }
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    var resolvedTags []*resolvedTag
    if !options.untrusted {
        translator := func(ctx context.Context, ref reference.NamedTagged) (reference.Canonical, error) {
            return TrustedReference(ctx, dockerCli, ref, nil)
        }
        // if there is a tar wrapper, the dockerfile needs to be replaced inside it
        if buildCtx != nil {
            // Wrap the tar archive to replace the Dockerfile entry with the rewritten
            // Dockerfile which uses trusted pulls.
            buildCtx = replaceDockerfileForContentTrust(ctx, buildCtx, relDockerfile, translator, &resolvedTags)
        } else if dockerfileCtx != nil {
            // if there was not archive context still do the possible replacements in Dockerfile
            newDockerfile, _, err := rewriteDockerfileFromForContentTrust(ctx, dockerfileCtx, translator)
            if err != nil {
                return err
            }
            dockerfileCtx = ioutil.NopCloser(bytes.NewBuffer(newDockerfile))
        }
    }

    if options.compress {
        buildCtx, err = build.Compress(buildCtx)
        if err != nil {
            return err
        }
    }
    // Setup an upload progress bar
    progressOutput := streamformatter.NewProgressOutput(progBuff)
    if !dockerCli.Out().IsTerminal() {
        progressOutput = &lastProgressOutput{output: progressOutput}
    }

    // if up to this point nothing has set the context then we must have another
    // way for sending it(streaming) and set the context to the Dockerfile
    if dockerfileCtx != nil && buildCtx == nil {
        buildCtx = dockerfileCtx
    }

    var body io.Reader
    if buildCtx != nil {
        body = progress.NewProgressReader(buildCtx, progressOutput, 0, "", "Sending build context to Docker daemon")
    }

    configFile := dockerCli.ConfigFile()
    creds, _ := configFile.GetAllCredentials()
    authConfigs := make(map[string]types.AuthConfig, len(creds))
    for k, auth := range creds {
        authConfigs[k] = types.AuthConfig(auth)
    }
    buildOptions := imageBuildOptions(dockerCli, options)
    buildOptions.Version = types.BuilderV1
    buildOptions.Dockerfile = relDockerfile
    buildOptions.AuthConfigs = authConfigs
    buildOptions.RemoteContext = remote
    response, err := dockerCli.Client().ImageBuild(ctx, body, buildOptions)
    if err != nil {
        if options.quiet {
            fmt.Fprintf(dockerCli.Err(), "%s", progBuff)
        }
        cancel()
        return err
    }
    defer response.Body.Close()

    imageID := ""
    aux := func(msg jsonmessage.JSONMessage) {
        var result types.BuildResult
        if err := json.Unmarshal(*msg.Aux, &result); err != nil {
            fmt.Fprintf(dockerCli.Err(), "Failed to parse aux message: %s", err)
        } else {
            imageID = result.ID
        }
    }

    err = jsonmessage.DisplayJSONMessagesStream(response.Body, buildBuff, dockerCli.Out().FD(), dockerCli.Out().IsTerminal(), aux)
    if err != nil {
        if jerr, ok := err.(*jsonmessage.JSONError); ok {
            // If no error code is set, default to 1
            if jerr.Code == 0 {
                jerr.Code = 1
            }
            if options.quiet {
                fmt.Fprintf(dockerCli.Err(), "%s%s", progBuff, buildBuff)
            }
            return cli.StatusError{Status: jerr.Message, StatusCode: jerr.Code}
        }
        return err
    }
    // Windows: show error message about modified file permissions if the
    // daemon isn't running Windows.
    if response.OSType != "windows" && runtime.GOOS == "windows" && !options.quiet {
        fmt.Fprintln(dockerCli.Out(), "SECURITY WARNING: You are building a Docker "+
            "image from Windows against a non-Windows Docker host. All files and "+
            "directories added to build context will have '-rwxr-xr-x' permissions. "+
            "It is recommended to double check and reset permissions for sensitive "+
            "files and directories.")
    }

    // Everything worked so if -q was provided the output from the daemon
    // should be just the image ID and we'll print that to stdout.
    if options.quiet {
        imageID = fmt.Sprintf("%s", buildBuff)
        _, _ = fmt.Fprint(dockerCli.Out(), imageID)
    }

    if options.imageIDFile != "" {
        if imageID == "" {
            return errors.Errorf("Server did not provide an image ID. Cannot write %s", options.imageIDFile)
        }
        if err := ioutil.WriteFile(options.imageIDFile, []byte(imageID), 0666); err != nil {
            return err
        }
    }
    if !options.untrusted {
        // Since the build was successful, now we must tag any of the resolved
        // images from the above Dockerfile rewrite.
        for _, resolved := range resolvedTags {
            if err := TagTrusted(ctx, dockerCli, resolved.digestRef, resolved.tagRef); err != nil {
                return err
            }
        }
    }

    return nil
}
{{< /highlight >}}

# command.BuildKitEnabled
{{< highlight go "linenos=inline" >}}
// github.com/docker/cli/cli/command/cli.go
// BuildKitEnabled returns whether buildkit is enabled either through a daemon setting
// or otherwise the client-side DOCKER_BUILDKIT environment variable
func BuildKitEnabled(si ServerInfo) (bool, error) {
    buildkitEnabled := si.BuildkitVersion == types.BuilderBuildKit
    if buildkitEnv := os.Getenv("DOCKER_BUILDKIT"); buildkitEnv != "" {
        var err error
        buildkitEnabled, err = strconv.ParseBool(buildkitEnv)
        if err != nil {
            return false, errors.Wrap(err, "DOCKER_BUILDKIT environment variable expects boolean value")
        }
    }
    return buildkitEnabled, nil
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

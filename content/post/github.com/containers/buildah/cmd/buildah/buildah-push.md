---
title: "buildah push"
date: 2020-02-10T18:14:26+08:00
draft: true
categories: ["技术"]
tags: ["buildah"]
---
# 简述
<!--more-->
# init
init主要初始化pushCommand
{{< highlight go "linenos=inline" >}}
func init() {
    var (
        opts            pushOptions
        pushDescription = fmt.Sprintf(`
  Pushes an image to a specified location.
  The Image "DESTINATION" uses a "transport":"details" format. If not specified, will reuse source IMAGE as DESTINATION.
  Supported transports:
  %s
  See buildah-push(1) section "DESTINATION" for the expected format
`, getListOfTransports())
    )

    pushCommand := &cobra.Command{
        Use:   "push",
        Short: "Push an image to a specified destination",
        Long:  pushDescription,
        RunE: func(cmd *cobra.Command, args []string) error {
            return pushCmd(cmd, args, opts)
        },
        Example: `buildah push imageID docker://registry.example.com/repository:tag
  buildah push imageID docker-daemon:image:tagi
  buildah push imageID oci:/path/to/layout:image:tag`,
    }
    pushCommand.SetUsageTemplate(UsageTemplate())

    flags := pushCommand.Flags()
    flags.SetInterspersed(false)
    flags.StringVar(&opts.authfile, "authfile", buildahcli.GetDefaultAuthFile(), "path of the authentication file. Use REGISTRY_AUTH_FILE environment variable to overrid    e")
    ......
    flags.BoolVarP(&opts.quiet, "quiet", "q", false, "don't output progress information when pushing images")
    flags.BoolVarP(&opts.removeSignatures, "remove-signatures", "", false, "don't copy signatures when pushing image")
    flags.StringVar(&opts.signBy, "sign-by", "", "sign the image using a GPG key with the specified `FINGERPRINT`")
    flags.StringVar(&opts.signaturePolicy, "signature-policy", "", "`pathname` of signature policy file (not usually used)")
    if err := flags.MarkHidden("signature-policy"); err != nil {
        panic(fmt.Sprintf("error marking signature-policy as hidden: %v", err))
    }
    flags.BoolVar(&opts.tlsVerify, "tls-verify", true, "require HTTPS and verify certificates when accessing the registry")
    if err := flags.MarkHidden("blob-cache"); err != nil {
        panic(fmt.Sprintf("error marking blob-cache as hidden: %v", err))
    }

    rootCmd.AddCommand(pushCommand)
}
{{< /highlight >}}

# pushCmd
{{< highlight go "linenos=inline" >}}
func pushCmd(c *cobra.Command, args []string, iopts pushOptions) error {
    var src, destSpec string

    if err := buildahcli.VerifyFlagsArgsOrder(args); err != nil {
        return err
    }
    if err := buildahcli.CheckAuthFile(iopts.authfile); err != nil {
        return err
    }

    switch len(args) {
    case 0:
        return errors.New("At least a source image ID must be specified")
    case 1:
        src = args[0]
        destSpec = src
        logrus.Debugf("Destination argument not specified, assuming the same as the source: %s", destSpec)
    case 2:
        src = args[0]
        destSpec = args[1]
        if src == "" {
            return errors.Errorf(`Invalid image name "%s"`, args[0])
        }
    default:
        return errors.New("Only two arguments are necessary to push: source and destination")
    }
    compress := imagebuildah.Gzip
    if iopts.disableCompression {
        compress = imagebuildah.Uncompressed
    }

    store, err := getStore(c)
    if err != nil {
        return err
    }

    dest, err := alltransports.ParseImageName(destSpec)
    // add the docker:// transport to see if they neglected it.
    if err != nil {
        destTransport := strings.Split(destSpec, ":")[0]
        if t := transports.Get(destTransport); t != nil {
            return err
        }

        if strings.Contains(destSpec, "://") {
            return err
        }

        destSpec = "docker://" + destSpec
        dest2, err2 := alltransports.ParseImageName(destSpec)
        if err2 != nil {
            return err
        }
        dest = dest2
        logrus.Debugf("Assuming docker:// as the transport method for DESTINATION: %s", destSpec)
    }

    systemContext, err := parse.SystemContextFromOptions(c)
    if err != nil {
        return errors.Wrapf(err, "error building system context")
    }

    var manifestType string
    if iopts.format != "" {
        switch iopts.format {
        case "oci":
            manifestType = imgspecv1.MediaTypeImageManifest
        case "v2s1":
            manifestType = manifest.DockerV2Schema1SignedMediaType
        case "v2s2", "docker":
            manifestType = manifest.DockerV2Schema2MediaType
        default:
            return fmt.Errorf("unknown format %q. Choose on of the supported formats: 'oci', 'v2s1', or 'v2s2'", iopts.format)
        }
    }

    options := buildah.PushOptions{
        Compression:         compress,
        ManifestType:        manifestType,
        SignaturePolicyPath: iopts.signaturePolicy,
        Store:               store,
        SystemContext:       systemContext,
        BlobDirectory:       iopts.blobCache,
        RemoveSignatures:    iopts.removeSignatures,
        SignBy:              iopts.signBy,
    }
    if !iopts.quiet {
        options.ReportWriter = os.Stderr
    }

    ref, digest, err := buildah.Push(getContext(), src, dest, options)
    if err != nil {
        return util.GetFailureCause(err, errors.Wrapf(err, "error pushing image %q to %q", src, destSpec))
    }
    if ref != nil {
        logrus.Debugf("pushed image %q with digest %s", ref, digest.String())
    } else {
        logrus.Debugf("pushed image with digest %s", digest.String())
    }

    logrus.Debugf("Successfully pushed %s with digest %s", transports.ImageName(dest), digest.String())

    if iopts.digestfile != "" {
        if err = ioutil.WriteFile(iopts.digestfile, []byte(digest.String()), 0644); err != nil {
            return util.GetFailureCause(err, errors.Wrapf(err, "failed to write digest to file %q", iopts.digestfile))
        }
    }

    return nil
}
{{< /highlight >}}

# pushOptions
{{< highlight go "linenos=inline" >}}
type pushOptions struct {
    authfile           string
    blobCache          string
    certDir            string
    creds              string
    digestfile         string
    disableCompression bool
    format             string
    quiet              bool
    removeSignatures   bool
    signaturePolicy    string
    signBy             string
    tlsVerify          bool
}
{{< /highlight >}}

# getListOfTransports
{{< highlight go "linenos=inline" >}}
// github.com/containers/buildah/cmd/buildah/push.go
// getListOfTransports gets the transports supported from the image library
// and strips of the "tarball" transport from the string of transports returned
func getListOfTransports() string {
    allTransports := strings.Join(transports.ListNames(), ",")
    return strings.Replace(allTransports, ",tarball", "", 1)
}
{{< /highlight >}}

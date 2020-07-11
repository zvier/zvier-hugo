---
title: "buildkit build flow"
date: 2020-03-01T08:18:26+08:00
draft: true
categories: ["技术"]
tags: ["buildkit"]
---
<!--more-->
# 命令行
{{< highlight go "linenos=inline" >}}
sudo bin/buildctl build  --frontend=dockerfile.v0 --local context=test --local dockerfile=test/ --output type=local,dest=images/test
{{< /highlight >}}

# 入口buildCommand
{{< highlight go "linenos=inline" >}}
// github.com/moby/buildkit/cmd/buildctl/build.go
var buildCommand = cli.Command{
    Name:    "build",
    Aliases: []string{"b"},
    ...
    Action: buildAction,
    ...
}
{{< /highlight >}}

# buildAction
{{< highlight go "linenos=inline" >}}
func buildAction(clicontext *cli.Context) error {
    // bccommon "github.com/moby/buildkit/cmd/buildctl/common"
    c, err := bccommon.ResolveClient(clicontext)
    if err != nil {
        return err
    }

    traceFile, err := openTraceFile(clicontext)
    if err != nil {
        return err
    }
    var traceEnc *json.Encoder
    if traceFile != nil {
        defer traceFile.Close()
        traceEnc = json.NewEncoder(traceFile)

        logrus.Infof("tracing logs to %s", traceFile.Name())
    }

    // "github.com/moby/buildkit/session/auth/authprovider"
    attachable := []session.Attachable{authprovider.NewDockerAuthProvider(os.Stderr)}

    if ssh := clicontext.StringSlice("ssh"); len(ssh) > 0 {
        configs, err := build.ParseSSH(ssh)
        if err != nil {
            return err
        }
        sp, err := sshprovider.NewSSHAgentProvider(configs)
        if err != nil {
            return err
        }
         attachable = append(attachable, sp)
     }
     if secrets := clicontext.StringSlice("secret"); len(secrets) > 0 {
         secretProvider, err := build.ParseSecret(secrets)
         if err != nil {
             return err
         }
         attachable = append(attachable, secretProvider)
     }

     allowed, err := build.ParseAllow(clicontext.StringSlice("allow"))
     if err != nil {
         return err
     }

     var exports []client.ExportEntry
     if legacyExporter := clicontext.String("exporter"); legacyExporter != "" {
         logrus.Warnf("--exporter <exporter> is deprecated. Please use --output type=<exporter>[,<opt>=<optval>] instead.")
         if len(clicontext.StringSlice("output")) > 0 {
             return errors.New("--exporter cannot be used with --output")
         }
         exports, err = build.ParseLegacyExporter(clicontext.String("exporter"), clicontext.StringSlice("exporter-opt"))
     } else {
         exports, err = build.ParseOutput(clicontext.StringSlice("output"))
     }
     if err != nil {
         return err
     }
     cacheExports, err := build.ParseExportCache(clicontext.StringSlice("export-cache"), clicontext.StringSlice("export-cache-opt"))
     if err != nil {
         return err
     }
     cacheImports, err := build.ParseImportCache(clicontext.StringSlice("import-cache"))
     if err != nil {
        return err
    }

    ch := make(chan *client.SolveStatus)
    // 创建一个errgroup.Group
    // func WithContext(ctx context.Context) (*Group, context.Context)
    eg, ctx := errgroup.WithContext(bccommon.CommandContext(clicontext))

    solveOpt := client.SolveOpt{
        Exports: exports,
        // LocalDirs is set later
        Frontend: clicontext.String("frontend"),
        // FrontendAttrs is set later
        CacheExports:        cacheExports,
        CacheImports:        cacheImports,
        Session:             attachable,
        AllowedEntitlements: allowed,
    }

    solveOpt.FrontendAttrs, err = build.ParseOpt(clicontext.StringSlice("opt"), clicontext.StringSlice("frontend-opt"))
    if err != nil {
        return errors.Wrap(err, "invalid opt")
    }

    solveOpt.LocalDirs, err = build.ParseLocal(clicontext.StringSlice("local"))
    if err != nil {
        return errors.Wrap(err, "invalid local")
    }

    var def *llb.Definition
    if clicontext.String("frontend") == "" {
        if fi, _ := os.Stdin.Stat(); (fi.Mode() & os.ModeCharDevice) != 0 {
            return errors.Errorf("please specify --frontend or pipe LLB definition to stdin")
        }
        def, err = read(os.Stdin, clicontext)
        if err != nil {
            return err
        }
        if len(def.Def) == 0 {
            return errors.Errorf("empty definition sent to build. Specify --frontend instead?")
        }
    } else {
        if clicontext.Bool("no-cache") {
            solveOpt.FrontendAttrs["no-cache"] = ""
        }
    }

    eg.Go(func() error {
        resp, err := c.Solve(ctx, def, solveOpt, ch)
        if err != nil {
            return err
        }
        for k, v := range resp.ExporterResponse {
            logrus.Debugf("exporter response: %s=%s", k, v)
        }
        return err
    })

   displayCh := ch
   if traceEnc != nil {
       // "github.com/moby/buildkit/client"
       displayCh = make(chan *client.SolveStatus)
       eg.Go(func() error {
           defer close(displayCh)
           for s := range ch {
               if err := traceEnc.Encode(s); err != nil {
                   logrus.Error(err)
               }
               displayCh <- s
           }
           return nil
       })
   }

   eg.Go(func() error {
       var c console.Console
       progressOpt := clicontext.String("progress")

       // progressOpt: auto
       switch progressOpt {
       case "auto", "tty":
           cf, err := console.ConsoleFromFile(os.Stderr)
           if err != nil && progressOpt == "tty" {
               return err
           }
           c = cf
       case "plain":
       default:
           return errors.Errorf("invalid progress value : %s", progressOpt)
       }

       // not using shared context to not disrupt display but let is finish reporting errors
       return progressui.DisplaySolveStatus(context.TODO(), "", c, os.Stderr, displayCh)
   })

   return eg.Wait()
}
{{< /highlight >}}

## 引用
[bccommon.ResolveClient](http://www.zvier.top/post/moby-buildkit-cmd-buildctl-common-common/#resolveclient)

# errgroup.Group
A Group is a collection of goroutines working on subtasks that are part of the same overall task.

A zero Group is valid and does not cancel on error.
{{< highlight go "linenos=inline" >}}
type Group struct {
    // contains filtered or unexported fields
}
{{< /highlight >}}

# errgroup.WithContext
WithContext returns a new Group and an associated Context derived from ctx.

The derived Context is canceled the first time a function passed to Go returns a non-nil error or the first time Wait returns, whichever occurs first.
{{< highlight go "linenos=inline" >}}
func WithContext(ctx context.Context) (*Group, context.Context)
{{< /highlight >}}

# Group.Go
Go calls the given function in a new goroutine.

The first call to return a non-nil error cancels the group; its error will be returned by Wait.
{{< highlight go "linenos=inline" >}}
func (g *Group) Go(f func() error)
{{< /highlight >}}

# Group.Wait
Wait blocks until all function calls from the Go method have returned, then returns the first non-nil error (if any) from them.
{{< highlight go "linenos=inline" >}}
func (g *Group) Wait() error
{{< /highlight >}}

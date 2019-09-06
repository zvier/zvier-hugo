---
title: "containerd源码解析(4)——LoadPlugins"
date: 2019-09-01T20:12:28+08:00
draft: true
categories: [""]
tags: ["containerd"]
---
# 概述
在server.New创建和初始化server时，会调用LoadPlugins来加载所需要的plugin
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/services/server/server.go
func New(ctx context.Context, config *srvconfig.Config) (*Server, error) {
    ...
   plugins, err := LoadPlugins(ctx, config)
    ...
}
{{< /highlight >}}

# LoadPlugins
LoadPlugins函数主要是加载所有的插件到containerd，并形成一个有向图

{{< highlight go "linenos=inline" >}}
import "github.com/containerd/containerd/plugin"
// github.com/containerd/containerd/services/server/server.go
// LoadPlugins loads all plugins into containerd and generates an ordered graph of all plugins.
func LoadPlugins(ctx context.Context, config *srvconfig.Config) ([]*plugin.Registration, error) {
    // load all plugins into containerd
    path := config.PluginDir
    if path == "" {
        path = filepath.Join(config.Root, "plugins")
    }
    if err := plugin.Load(path); err != nil {
        return nil, err
    }
    ...
}
{{< /highlight >}}

## plugin.Load
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/plugin/plugin.go
// Load loads all plugins at the provided path into containerd
func Load(path string) (err error) {
    defer func() {
        if v := recover(); v != nil {
            rerr, ok := v.(error)
            if !ok {
                rerr = fmt.Errorf("%s", v)
            }
            err = rerr
        }
    }()
    return loadPlugins(path)
}
{{< /highlight >}}

## loadPlugins
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/plugin/plugin_go18.go
// loadPlugins loads all plugins for the OS and Arch
// that containerd is built for inside the provided path
func loadPlugins(path string) error {
    abs, err := filepath.Abs(path)
    if err != nil {
        return err
    }
    pattern := filepath.Join(abs, fmt.Sprintf(
        "*-%s-%s.%s",
        runtime.GOOS,
        runtime.GOARCH,
        getLibExt(),
    ))
    libs, err := filepath.Glob(pattern)
    if err != nil {
        return err
    }
    for _, lib := range libs {
        if _, err := plugin.Open(lib); err != nil {
            return err
        }
    }
    return nil
}
{{< /highlight >}}

## plugin.Open
Open opens a Go plugin. If a path has already been opened, then the existing *Plugin is returned. It is safe for concurrent use by multiple goroutines.
{{< highlight go "linenos=inline" >}}
func Open(path string) (*Plugin, error)
{{< /highlight >}}

Plugin is a loaded Go plugin.
{{< highlight go "linenos=inline" >}}
type Plugin struct {
    // contains filtered or unexported fields
}
{{< /highlight >}}

## Glob
package filepath
Glob函数返回所有匹配模式匹配字符串pattern的文件或者nil（如果没有匹配的文件）。pattern的语法和Match函数相同。pattern可以描述多层的名字，如/usr/*/bin/ed（假设路径分隔符是'/'）。
{{< highlight go "linenos=inline" >}}
func Glob(pattern string) (matches []string, err error)
{{< /highlight >}}

## plugin.Register
{{< highlight go "linenos=inline" >}}
func LoadPlugins(ctx context.Context, config *srvconfig.Config) ([]*plugin.Registration, error) {
    ...
    // load additional plugins that don't automatically register themselves
    plugin.Register(&plugin.Registration{
        Type: plugin.ContentPlugin,
        ID:   "content",
        InitFn: func(ic *plugin.InitContext) (interface{}, error) {
            ic.Meta.Exports["root"] = ic.Root
            return local.NewStore(ic.Root)
        },
    })
    plugin.Register(&plugin.Registration{
        Type: plugin.MetadataPlugin,
        ID:   "bolt",
        Requires: []plugin.Type{
            plugin.ContentPlugin,
            plugin.SnapshotPlugin,
        },
        Config: &srvconfig.BoltConfig{
            ContentSharingPolicy: srvconfig.SharingPolicyShared,
        },
        InitFn: func(ic *plugin.InitContext) (interface{}, error) {
            if err := os.MkdirAll(ic.Root, 0711); err != nil {
                return nil, err
            }
            cs, err := ic.Get(plugin.ContentPlugin)
            if err != nil {
                return nil, err
            }

            snapshottersRaw, err := ic.GetByType(plugin.SnapshotPlugin)
            if err != nil {
                return nil, err
            }

            snapshotters := make(map[string]snapshots.Snapshotter)
            for name, sn := range snapshottersRaw {
                sn, err := sn.Instance()
                if err != nil {
                    if !plugin.IsSkipPlugin(err) {
                        log.G(ic.Context).WithError(err).
                            Warnf("could not use snapshotter %v in metadata plugin", name)
                    }
                    continue
                }
                snapshotters[name] = sn.(snapshots.Snapshotter)
            }

            shared := true
            ic.Meta.Exports["policy"] = srvconfig.SharingPolicyShared
            if cfg, ok := ic.Config.(*srvconfig.BoltConfig); ok {
                if cfg.ContentSharingPolicy != "" {
                    if err := cfg.Validate(); err != nil {
                        return nil, err
                    }
                    if cfg.ContentSharingPolicy == srvconfig.SharingPolicyIsolated {
                        ic.Meta.Exports["policy"] = srvconfig.SharingPolicyIsolated
                        shared = false
                    }
8                     log.L.WithField("policy", cfg.ContentSharingPolicy).Info("metadata content store policy set")
                }
            }

            path := filepath.Join(ic.Root, "meta.db")
            ic.Meta.Exports["path"] = path

            db, err := bolt.Open(path, 0644, nil)
            if err != nil {
                return nil, err
            }

            var dbopts []metadata.DBOpt
            if !shared {
                dbopts = append(dbopts, metadata.WithPolicyIsolated)
            }
            mdb := metadata.NewDB(db, cs.(content.Store), snapshotters, dbopts...)
            if err := mdb.Init(ic.Context); err != nil {
                return nil, err
            }
            return mdb, nil
        },
    })
    clients := &proxyClients{}
    for name, pp := range config.ProxyPlugins {
        var (
            t plugin.Type
            f func(*grpc.ClientConn) interface{}

            address = pp.Address
        )

        switch pp.Type {
        case string(plugin.SnapshotPlugin), "snapshot":
            t = plugin.SnapshotPlugin
            ssname := name
            f = func(conn *grpc.ClientConn) interface{} {
                return ssproxy.NewSnapshotter(ssapi.NewSnapshotsClient(conn), ssname)
            }

        case string(plugin.ContentPlugin), "content":
            t = plugin.ContentPlugin
            f = func(conn *grpc.ClientConn) interface{} {
                return csproxy.NewContentStore(csapi.NewContentClient(conn))
            }
        default:
            log.G(ctx).WithField("type", pp.Type).Warn("unknown proxy plugin type")
        }
        plugin.Register(&plugin.Registration{
            Type: t,
            ID:   name,
            InitFn: func(ic *plugin.InitContext) (interface{}, error) {
                ic.Meta.Exports["address"] = address
                conn, err := clients.getClient(address)
                if err != nil {
                    return nil, err
                }
                return f(conn), nil
            },
        })

    }

    filter := srvconfig.V2DisabledFilter
    if config.GetVersion() == 1 {
        filter = srvconfig.V1DisabledFilter
    }
    // return the ordered graph for plugins
    return plugin.Graph(filter(config.DisabledPlugins)), nil
}

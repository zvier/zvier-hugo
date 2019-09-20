---
title: "containerd-containerd-cmd-containerd-command-config"
date: 2019-09-08T18:30:30+08:00
draft: true
categories: [""]
tags: ["containerd"]
---

# 简述
<code>containerd config</code>子命令行，containerd配置相关，比如:
{{< highlight go "linenos=inline" >}}
containerd config default
{{< /highlight >}}
<!--more-->
将输入containerd的配置信息

# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/containerd/containerd/cmd/containerd/command/config.go
{{< /highlight >}}

# Config
用于存储server配置的数据结构
{{< highlight go "linenos=inline" >}}
import (
    srvconfig "github.com/containerd/containerd/services/server/config"
)
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// Config is a wrapper of server config for printing out.
type Config struct {
	*srvconfig.Config
	// Plugins overrides `Plugins map[string]toml.Primitive` in server config.
	Plugins map[string]interface{} `toml:"plugins"`
}
{{< /highlight >}}

## 引用说明
srvconfig.Config

# Config.WriteTo
将配置文件信息写入到io.Writer实例
{{< highlight go "linenos=inline" >}}
import "github.com/BurntSushi/toml"
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// WriteTo marshals the config to the provided writer
func (c *Config) WriteTo(w io.Writer) (int64, error) {
	return 0, toml.NewEncoder(w).Encode(c)
}
{{< /highlight >}}

# outputConfig
{{< highlight go "linenos=inline" >}}
import (
	gocontext "context"
	"github.com/containerd/containerd/pkg/timeout"
	"github.com/containerd/containerd/services/server"
)
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
func outputConfig(cfg *srvconfig.Config) error {
	config := &Config{
		Config: cfg,
	}

	plugins, err := server.LoadPlugins(gocontext.Background(), config.Config)
	if err != nil {
		return err
	}
	if len(plugins) != 0 {
		config.Plugins = make(map[string]interface{})
		for _, p := range plugins {
			if p.Config == nil {
				continue
			}

			pc, err := config.Decode(p)
			if err != nil {
				return err
			}

			config.Plugins[p.URI()] = pc
		}
	}

	timeouts := timeout.All()
	config.Timeouts = make(map[string]string)
	for k, v := range timeouts {
		config.Timeouts[k] = v.String()
	}

	// for the time being, keep the defaultConfig's version set at 1 so that
	// when a config without a version is loaded from disk and has no version
	// set, we assume it's a v1 config.  But when generating new configs via
	// this command, generate the v2 config
	config.Config.Version = 2

	// remove overridden Plugins type to avoid duplication in output
	config.Config.Plugins = nil

	_, err = config.WriteTo(os.Stdout)
	return err
}
{{< /highlight >}}

# configCommand
定义containerd的子命令行<code>config</code>，<code>containerd
config</code>又有两个子命令行:  
1. containerd config default  
2. containerd config dump  

{{< highlight go "linenos=inline" >}}
import (
	"github.com/urfave/cli"
)
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
var configCommand = cli.Command{
	Name:  "config",
	Usage: "information on the containerd config",
	Subcommands: []cli.Command{
		{
			Name:  "default",
			Usage: "see the output of the default config",
			Action: func(context *cli.Context) error {
				return outputConfig(defaultConfig())
			},
		},
		{
			Name:  "dump",
			Usage: "see the output of the final main config with imported in subconfig files",
			Action: func(context *cli.Context) error {
				config := defaultConfig()
				if err := srvconfig.LoadConfig(context.GlobalString("config"), config); err != nil && !os.IsNotExist(err) {
					return err
				}

				return outputConfig(config)
			},
		},
	},
}
{{< /highlight >}}

## 引用说明
defaultConfig
srvconfig.LoadConfig

---
title: "containerd-containerd-services-server-config-config"
date: 2019-09-11T21:28:04+08:00
draft: true
categories: [""]
tags: ["containerd"]
---
# 简述
containerd server配置相关
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/containerd/containerd/services/server/config/config.go
{{< /highlight >}}

package config

import (
	"path/filepath"
	"strings"

	"github.com/imdario/mergo"
	"github.com/pkg/errors"

	"github.com/containerd/containerd/errdefs"
	"github.com/containerd/containerd/plugin"
)
# Config
containerd server的配置
{{< highlight go "linenos=inline" >}}
import (
	"github.com/BurntSushi/toml"
)
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
// Config provides containerd configuration data for the server
type Config struct {
	// Version of the config file
	Version int `toml:"version"`
	// Root is the path to a directory where containerd will store persistent data
	Root string `toml:"root"`
	// State is the path to a directory where containerd will store transient data
	State string `toml:"state"`
	// PluginDir is the directory for dynamic plugins to be stored
	PluginDir string `toml:"plugin_dir"`
	// GRPC configuration settings
	GRPC GRPCConfig `toml:"grpc"`
	// TTRPC configuration settings
	TTRPC TTRPCConfig `toml:"ttrpc"`
	// Debug and profiling settings
	Debug Debug `toml:"debug"`
	// Metrics and monitoring settings
	Metrics MetricsConfig `toml:"metrics"`
	// DisabledPlugins are IDs of plugins to disable. Disabled plugins won't be
	// initialized and started.
	DisabledPlugins []string `toml:"disabled_plugins"`
	// RequiredPlugins are IDs of required plugins. Containerd exits if any
	// required plugin doesn't exist or fails to be initialized or started.
	RequiredPlugins []string `toml:"required_plugins"`
	// Plugins provides plugin specific configuration for the initialization of a plugin
	Plugins map[string]toml.Primitive `toml:"plugins"`
	// OOMScore adjust the containerd's oom score
	OOMScore int `toml:"oom_score"`
	// Cgroup specifies cgroup information for the containerd daemon process
	Cgroup CgroupConfig `toml:"cgroup"`
	// ProxyPlugins configures plugins which are communicated to over GRPC
	ProxyPlugins map[string]ProxyPlugin `toml:"proxy_plugins"`
	// Timeouts specified as a duration
	Timeouts map[string]string `toml:"timeouts"`
	// Imports are additional file path list to config files that can overwrite main config file fields
	Imports []string `toml:"imports"`

	StreamProcessors map[string]StreamProcessor `toml:"stream_processors"`
}
{{< /highlight >}}
## toml.Primitive
toml包提供了通过反摄编码和解码TOML配置文件的工具，支持使用Primitive类型延迟解码，支持使用MetaData类型查询TOML文档中的键集  
{{< highlight go "linenos=inline" >}}
import "github.com/BurntSushi/toml"
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
type Primitive struct {
    // contains filtered or unexported fields
}
{{< /highlight >}}

> Primitive is a TOML value that hasn't been decoded into a Go value. When using the various `Decode*` functions, the type `Primitive` may be given to any value, and its decoding will be delayed.  
> A `Primitive` value can be decoded using the `PrimitiveDecode` function. The underlying representation of a `Primitive` value is subject to change. Do not rely on it.  N.B. Primitive values are still parsed, so using them will only avoid the overhead of reflection(避免反射的开销). They can be useful when you don't know the exact type of TOML data until run time.

# StreamProcessor
{{< highlight go "linenos=inline" >}}
// StreamProcessor provides configuration for diff content processors
type StreamProcessor struct {
	// Accepts specific media-types
	Accepts []string `toml:"accepts"`
	// Returns the media-type
	Returns string `toml:"returns"`
	// Path or name of the binary
	Path string `toml:"path"`
	// Args to the binary
	Args []string `toml:"args"`
}
{{< /highlight >}}

# Config.GetVersion
{{< highlight go "linenos=inline" >}}
// GetVersion returns the config file's version
func (c *Config) GetVersion() int {
	if c.Version == 0 {
		return 1
	}
	return c.Version
}
{{< /highlight >}}

# Config.ValidateV2
{{< highlight go "linenos=inline" >}}
// ValidateV2 validates the config for a v2 file
func (c *Config) ValidateV2() error {
	if c.GetVersion() != 2 {
		return nil
	}
	for _, p := range c.DisabledPlugins {
		if len(strings.Split(p, ".")) < 4 {
			return errors.Errorf("invalid disabled plugin URI %q expect io.containerd.x.vx", p)
		}
	}
	for _, p := range c.RequiredPlugins {
		if len(strings.Split(p, ".")) < 4 {
			return errors.Errorf("invalid required plugin URI %q expect io.containerd.x.vx", p)
		}
	}
	for p := range c.Plugins {
		if len(strings.Split(p, ".")) < 4 {
			return errors.Errorf("invalid plugin key URI %q expect io.containerd.x.vx", p)
		}
	}
	return nil
}
{{< /highlight >}}

# GRPCConfig
{{< highlight go "linenos=inline" >}}
// GRPCConfig provides GRPC configuration for the socket
type GRPCConfig struct {
	Address        string `toml:"address"`
	TCPAddress     string `toml:"tcp_address"`
	TCPTLSCert     string `toml:"tcp_tls_cert"`
	TCPTLSKey      string `toml:"tcp_tls_key"`
	UID            int    `toml:"uid"`
	GID            int    `toml:"gid"`
	MaxRecvMsgSize int    `toml:"max_recv_message_size"`
	MaxSendMsgSize int    `toml:"max_send_message_size"`
}
{{< /highlight >}}

# TTRPCConfig
{{< highlight go "linenos=inline" >}}
// TTRPCConfig provides TTRPC configuration for the socket
type TTRPCConfig struct {
	Address string `toml:"address"`
	UID     int    `toml:"uid"`
	GID     int    `toml:"gid"`
}
{{< /highlight >}}

# Debug
{{< highlight go "linenos=inline" >}}
// Debug provides debug configuration
type Debug struct {
	Address string `toml:"address"`
	UID     int    `toml:"uid"`
	GID     int    `toml:"gid"`
	Level   string `toml:"level"`
}
{{< /highlight >}}

# MetricsConfig
{{< highlight go "linenos=inline" >}}
// MetricsConfig provides metrics configuration
type MetricsConfig struct {
	Address       string `toml:"address"`
	GRPCHistogram bool   `toml:"grpc_histogram"`
}
{{< /highlight >}}

# CgroupConfig
{{< highlight go "linenos=inline" >}}
// CgroupConfig provides cgroup configuration
type CgroupConfig struct {
	Path string `toml:"path"`
}
{{< /highlight >}}

# ProxyPlugin
{{< highlight go "linenos=inline" >}}
// ProxyPlugin provides a proxy plugin configuration
type ProxyPlugin struct {
	Type    string `toml:"type"`
	Address string `toml:"address"`
}
{{< /highlight >}}

# BoltConfig
{{< highlight go "linenos=inline" >}}
// BoltConfig defines the configuration values for the bolt plugin, which is
// loaded here, rather than back registered in the metadata package.
type BoltConfig struct {
	// ContentSharingPolicy sets the sharing policy for content between
	// namespaces.
	//
	// The default mode "shared" will make blobs available in all
	// namespaces once it is pulled into any namespace. The blob will be pulled
	// into the namespace if a writer is opened with the "Expected" digest that
	// is already present in the backend.
	//
	// The alternative mode, "isolated" requires that clients prove they have
	// access to the content by providing all of the content to the ingest
	// before the blob is added to the namespace.
	//
	// Both modes share backing data, while "shared" will reduce total
	// bandwidth across namespaces, at the cost of allowing access to any blob
	// just by knowing its digest.
	ContentSharingPolicy string `toml:"content_sharing_policy"`
}
{{< /highlight >}}

# 常量声明
{{< highlight go "linenos=inline" >}}
const (
	// SharingPolicyShared represents the "shared" sharing policy
	SharingPolicyShared = "shared"
	// SharingPolicyIsolated represents the "isolated" sharing policy
	SharingPolicyIsolated = "isolated"
)
{{< /highlight >}}

# BoltConfig.Validate
{{< highlight go "linenos=inline" >}}
// Validate validates if BoltConfig is valid
func (bc *BoltConfig) Validate() error {
	switch bc.ContentSharingPolicy {
	case SharingPolicyShared, SharingPolicyIsolated:
		return nil
	default:
		return errors.Wrapf(errdefs.ErrInvalidArgument, "unknown policy: %s", bc.ContentSharingPolicy)
	}
}
{{< /highlight >}}

# Config.Decode
{{< highlight go "linenos=inline" >}}
// Decode unmarshals a plugin specific configuration by plugin id
func (c *Config) Decode(p *plugin.Registration) (interface{}, error) {
	id := p.URI()
	if c.GetVersion() == 1 {
		id = p.ID
	}
	data, ok := c.Plugins[id]
	if !ok {
		return p.Config, nil
	}
	if err := toml.PrimitiveDecode(data, p.Config); err != nil {
		return nil, err
	}
	return p.Config, nil
}
{{< /highlight >}}

# LoadConfig
{{< highlight go "linenos=inline" >}}
// LoadConfig loads the containerd server config from the provided path
func LoadConfig(path string, out *Config) error {
	if out == nil {
		return errors.Wrapf(errdefs.ErrInvalidArgument, "argument out must not be nil")
	}

	var (
		loaded  = map[string]bool{}
		pending = []string{path}
	)

	for len(pending) > 0 {
		path, pending = pending[0], pending[1:]

		// Check if a file at the given path already loaded to prevent circular imports
		if _, ok := loaded[path]; ok {
			continue
		}

		config, err := loadConfigFile(path)
		if err != nil {
			return err
		}

		if err := mergeConfig(out, config); err != nil {
			return err
		}

		imports, err := resolveImports(path, config.Imports)
		if err != nil {
			return err
		}

		loaded[path] = true
		pending = append(pending, imports...)
	}

	// Fix up the list of config files loaded
	out.Imports = []string{}
	for path := range loaded {
		out.Imports = append(out.Imports, path)
	}

	return out.ValidateV2()
}
{{< /highlight >}}

# loadConfigFile
{{< highlight go "linenos=inline" >}}
// loadConfigFile decodes a TOML file at the given path
func loadConfigFile(path string) (*Config, error) {
	config := &Config{}
	_, err := toml.DecodeFile(path, &config)
	if err != nil {
		return nil, err
	}
	return config, nil
}
{{< /highlight >}}

# resolveImports
{{< highlight go "linenos=inline" >}}
// resolveImports resolves import strings list to absolute paths list:
// - If path contains *, glob pattern matching applied
// - Non abs path is relative to parent config file directory
// - Abs paths returned as is
func resolveImports(parent string, imports []string) ([]string, error) {
	var out []string

	for _, path := range imports {
		if strings.Contains(path, "*") {
			matches, err := filepath.Glob(path)
			if err != nil {
				return nil, err
			}

			out = append(out, matches...)
		} else {
			path = filepath.Clean(path)
			if !filepath.IsAbs(path) {
				path = filepath.Join(filepath.Dir(parent), path)
			}

			out = append(out, path)
		}
	}

	return out, nil
}
{{< /highlight >}}

#  mergeConfig
{{< highlight go "linenos=inline" >}}
// mergeConfig merges Config structs with the following rules:
// 'to'         'from'      'result'
// ""           "value"     "value"
// "value"      ""          "value"
// 1            0           1
// 0            1           1
// []{"1"}      []{"2"}     []{"1","2"}
// []{"1"}      []{}        []{"1"}
// Maps merged by keys, but values are replaced entirely.
func mergeConfig(to, from *Config) error {
	err := mergo.Merge(to, from, mergo.WithOverride, mergo.WithAppendSlice)
	if err != nil {
		return err
	}

	// Replace entire sections instead of merging map's values.
	for k, v := range from.Plugins {
		to.Plugins[k] = v
	}

	for k, v := range from.StreamProcessors {
		to.StreamProcessors[k] = v
	}

	for k, v := range from.ProxyPlugins {
		to.ProxyPlugins[k] = v
	}

	return nil
}
{{< /highlight >}}

# V1DisabledFilter
{{< highlight go "linenos=inline" >}}
// V1DisabledFilter matches based on ID
func V1DisabledFilter(list []string) plugin.DisableFilter {
	set := make(map[string]struct{}, len(list))
	for _, l := range list {
		set[l] = struct{}{}
	}
	return func(r *plugin.Registration) bool {
		_, ok := set[r.ID]
		return ok
	}
}
{{< /highlight >}}

# V2DisabledFilter
{{< highlight go "linenos=inline" >}}
// V2DisabledFilter matches based on URI
func V2DisabledFilter(list []string) plugin.DisableFilter {
	set := make(map[string]struct{}, len(list))
	for _, l := range list {
		set[l] = struct{}{}
	}
	return func(r *plugin.Registration) bool {
		_, ok := set[r.URI()]
		return ok
	}
}
{{< /highlight >}}

---
title: "client-go 源码分析二： kubeconfig"
date: 2020-07-05T09:24:19+08:00
draft: true
categories: ["技术"]
tags: ["k8s"]
---
<!--more-->
# 简述
kubeconfig就是用于管理访问kube-apiserver的配置信息，文件中存储了集群、用户、命名空间和身份验证等信息，相关的组件通过加载kubeconfig配置来连接访问kube-apiserver。

# 配置解析
默认情况下，kubeconfig默认放在$HOME/.kube/config路径下，形式如下：
{{< highlight go "linenos=inline" >}}
cat $HOME/.kube/config

apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ...
    server: https://host-ip:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: ...
    client-key-data: ...
{{< /highlight >}}

## clusters
数组，clusters为kubernetes集群相关的信息，包括kube-apiserver的地址，集群名，证书信息等。

## contexts
数组，contexts为集群用户信息，命名空间等，用于将请求发送到指定的集群。

## users
数组，users为集群用户身份验证的客户端凭据等

# kubeconfig使用示例
{{< highlight go "linenos=inline" >}}
package main

import (
	"fmt"

	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "/home/liuzekun/.kube/config")
	if err != nil {
		fmt.Println(err)
	}
	fmt.Printf("config is: %+v\n", config)
}
{{< /highlight >}}

## clientcmd.BuildConfigFromFlags
BuildConfigFromFlags是一个helper function，读取kubeconfig配置信息，并返回restclient.Config指针对象。代码如下：

{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/tools/clientcmd/client_config.go
// BuildConfigFromFlags is a helper function that builds configs from a master
// url or a kubeconfig filepath. These are passed in as command line flags for cluster
// components. Warnings should reflect this usage. If neither masterUrl or kubeconfigPath
// are passed in we fallback to inClusterConfig. If inClusterConfig fails, we fallback
// to the default config.
// BuildConfigFromFlags参数可以是masterUrl，也可以是配置的路径kubeconfigPath
func BuildConfigFromFlags(masterUrl, kubeconfigPath string) (*restclient.Config, error) {
    if kubeconfigPath == "" && masterUrl == "" {
        klog.Warningf("Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.")
        kubeconfig, err := restclient.InClusterConfig()
        if err == nil {
            return kubeconfig, nil
        }
        klog.Warning("error creating inClusterConfig, falling back to default config: ", err)
    }
    return NewNonInteractiveDeferredLoadingClientConfig(
        &ClientConfigLoadingRules{ExplicitPath: kubeconfigPath},
        &ConfigOverrides{ClusterInfo: clientcmdapi.Cluster{Server: masterUrl}}).ClientConfig()
}
{{< /highlight >}}

## rest.Config
Config存储了kubeconfig配置信息，用于初始化kubernetes的各种client
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/rest/config.go
// Config holds the common attributes that can be passed to a Kubernetes client on initialization.
type Config struct {
    // Host must be a host string, a host:port pair, or a URL to the base of the apiserver.
    // If a URL is given then the (optional) Path of that URL represents a prefix that must
    // be appended to all request URIs used to access the apiserver. This allows a frontend
    // proxy to easily relocate all of the apiserver endpoints.
    Host string
    // APIPath is a sub-path that points to an API root.
    APIPath string

    // ContentConfig contains settings that affect how objects are transformed when
    // sent to the server.
    ContentConfig

    // Server requires Basic authentication
    Username string
    Password string

    // Server requires Bearer authentication. This client will not attempt to use
    // refresh tokens for an OAuth2 flow.
    // TODO: demonstrate an OAuth2 compatible client.
    BearerToken string

    // Path to a file containing a BearerToken.
    // If set, the contents are periodically read.
    // The last successfully read value takes precedence over BearerToken.
    BearerTokenFile string

    // Impersonate is the configuration that RESTClient will use for impersonation.
    Impersonate ImpersonationConfig

    // Server requires plugin-specified authentication.
    AuthProvider *clientcmdapi.AuthProviderConfig

    // Callback to persist config for AuthProvider.
    AuthConfigPersister AuthProviderConfigPersister

    // Exec-based authentication provider.
    ExecProvider *clientcmdapi.ExecConfig

     // TLSClientConfig contains settings to enable transport layer security
     TLSClientConfig
     // UserAgent is an optional field that specifies the caller of this request.
     UserAgent string

     // DisableCompression bypasses automatic GZip compression requests to the
     // server.
     DisableCompression bool

     // Transport may be used for custom HTTP behavior. This attribute may not
     // be specified with the TLS client certificate options. Use WrapTransport
     // to provide additional per-server middleware behavior.
     Transport http.RoundTripper
     // WrapTransport will be invoked for custom HTTP behavior after the underlying
     // transport is initialized (either the transport created from TLSClientConfig,
     // Transport, or http.DefaultTransport). The config may layer other RoundTrippers
     // on top of the returned RoundTripper.
     //
     // A future release will change this field to an array. Use config.Wrap()
     // instead of setting this value directly.
     WrapTransport transport.WrapperFunc

     // QPS indicates the maximum QPS to the master from this client.
     // If it's zero, the created RESTClient will use DefaultQPS: 5
     QPS float32

     // Maximum burst for throttle.
     // If it's zero, the created RESTClient will use DefaultBurst: 10.
     Burst int

     // Rate limiter for limiting connections to the master from this client. If present overwrites QPS/Burst
     RateLimiter flowcontrol.RateLimiter

     // The maximum length of time to wait before giving up on a server request. A value of zero means no timeout.
     Timeout time.Duration
     // Dial specifies the dial function for creating unencrypted TCP connections.
     Dial func(ctx context.Context, network, address string) (net.Conn, error)

     // Proxy is the the proxy func to be used for all requests made by this
     // transport. If Proxy is nil, http.ProxyFromEnvironment is used. If Proxy
     // returns a nil *URL, no proxy is used.
     //
     // socks5 proxying does not currently support spdy streaming endpoints.
     Proxy func(*http.Request) (*url.URL, error)

     // Version forces a specific version to be used (if registered)
     // Do we need this?
     // Version string
}
{{< /highlight >}}

## ClientConfigLoadingRules
获取kubeconfig配置信息有两种方式，一种是从文件路径ClientConfigLoadingRules.ExplicitPath中读取，还一种是从环境变量KUBECONFIG中获取，即ClientConfigLoadingRules.Precedence，多个路径中的配置信息将汇总到kubeConfigFiles中。
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/tools/clientcmd/loader.go
// ClientConfigLoadingRules is an ExplicitPath and string slice of specific locations that are used for merging together a Config
// Callers can put the chain together however they want, but we'd recommend:
// EnvVarPathFiles if set (a list of files if set) OR the HomeDirectoryPath
// ExplicitPath is special, because if a user specifically requests a certain file be used and error is reported if this file is not present
type ClientConfigLoadingRules struct {
    // 配置的路径
    ExplicitPath string
    Precedence   []string

    // MigrationRules is a map of destination files to source files.  If a destination file is not present, then the source file is checked.
    // If the source file is present, then it is copied to the destination file BEFORE any further loading happens.
    MigrationRules map[string]string

    // DoNotResolvePaths indicates whether or not to resolve paths with respect to the originating files.  This is phrased as a negative so
    // that a default object that doesn't set this will usually get the behavior it wants.
    DoNotResolvePaths bool

    // DefaultClientConfig is an optional field indicating what rules to use to calculate a default configuration.
    // This should match the overrides passed in to ClientConfig loader.
    DefaultClientConfig ClientConfig

    // WarnIfAllMissing indicates whether the configuration files pointed by KUBECONFIG environment variable are present or not.
    // In case of missing files, it warns the user about the missing files.
    WarnIfAllMissing bool
}
{{< /highlight >}}

## ConfigOverrides
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/tools/clientcmd/overrides.go
// ConfigOverrides holds values that should override whatever information is pulled from the actual Config object.  You can't
// simply use an actual Config object, because Configs hold maps, but overrides are restricted to "at most one"
type ConfigOverrides struct {
    AuthInfo clientcmdapi.AuthInfo
    // ClusterDefaults are applied before the configured cluster info is loaded.
    // clientcmdapi "k8s.io/client-go/tools/clientcmd/api"
    ClusterDefaults clientcmdapi.Cluster
    ClusterInfo     clientcmdapi.Cluster
    Context         clientcmdapi.Context
    CurrentContext  string
    Timeout         string
}
{{< /highlight >}}

## Cluster
Cluster结构用于存储与kubernetes集群通信交互的信息
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/tools/clientcmd/api/types.go
// Cluster contains information about how to communicate with a kubernetes cluster
type Cluster struct {
    // LocationOfOrigin indicates where this object came from.  It is used for round tripping config post-merge, but never serialized.
    // +k8s:conversion-gen=false
    LocationOfOrigin string
    // Server is the address of the kubernetes cluster (https://hostname:port).
    Server string `json:"server"`
    // TLSServerName is used to check server certificate. If TLSServerName is empty, the hostname used to contact the server is used.
    // +optional
    TLSServerName string `json:"tls-server-name,omitempty"`
    // InsecureSkipTLSVerify skips the validity check for the server's certificate. This will make your HTTPS connections insecure.
    // +optional
    InsecureSkipTLSVerify bool `json:"insecure-skip-tls-verify,omitempty"`
    // CertificateAuthority is the path to a cert file for the certificate authority.
    // +optional
    CertificateAuthority string `json:"certificate-authority,omitempty"`
    // CertificateAuthorityData contains PEM-encoded certificate authority certificates. Overrides CertificateAuthority
    // +optional
    CertificateAuthorityData []byte `json:"certificate-authority-data,omitempty"`
    // ProxyURL is the URL to the proxy to be used for all requests made by this
    // client. URLs with "http", "https", and "socks5" schemes are supported.  If
    // this configuration is not provided or the empty string, the client
    // attempts to construct a proxy configuration from http_proxy and
    // https_proxy environment variables. If these environment variables are not
    // set, the client does not attempt to proxy requests.
    //
    // socks5 proxying does not currently support spdy streaming endpoints (exec,
    // attach, port forward).
    // +optional
    ProxyURL string `json:"proxy-url,omitempty"`
    // Extensions holds additional information. This is useful for extenders so that reads and writes don't clobber unknown fields
    // +optional
    Extensions map[string]runtime.Object `json:"extensions,omitempty"`
}
{{< /highlight >}}

## NewNonInteractiveDeferredLoadingClientConfig
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/tools/clientcmd/merged_client_builder.go
// NewNonInteractiveDeferredLoadingClientConfig creates a ConfigClientClientConfig using the passed context name
func NewNonInteractiveDeferredLoadingClientConfig(loader ClientConfigLoader, overrides *ConfigOverrides) ClientConfig {
    return &DeferredLoadingClientConfig{loader: loader, overrides: overrides, icc: &inClusterClientConfig{overrides: overrides}}
}
{{< /highlight >}}

## DeferredLoadingClientConfig
{{< highlight go "linenos=inline" >}}
// DeferredLoadingClientConfig is a ClientConfig interface that is backed by a client config loader.
// It is used in cases where the loading rules may change after you've instantiated them and you want to be sure that
// the most recent rules are used.  This is useful in cases where you bind flags to loading rule parameters before
// the parse happens and you want your calling code to be ignorant of how the values are being mutated to avoid
// passing extraneous information down a call stack
type DeferredLoadingClientConfig struct {
    loader         ClientConfigLoader
    overrides      *ConfigOverrides
    fallbackReader io.Reader

    clientConfig ClientConfig
    loadingLock  sync.Mutex

    // provided for testing
    icc InClusterConfig
}
{{< /highlight >}}

## DeferredLoadingClientConfig
{{< highlight go "linenos=inline" >}}
// DeferredLoadingClientConfig is a ClientConfig interface that is backed by a client config loader.
// It is used in cases where the loading rules may change after you've instantiated them and you want to be sure that
// the most recent rules are used.  This is useful in cases where you bind flags to loading rule parameters before
// the parse happens and you want your calling code to be ignorant of how the values are being mutated to avoid
// passing extraneous information down a call stack
type DeferredLoadingClientConfig struct {
    loader         ClientConfigLoader
    overrides      *ConfigOverrides
    fallbackReader io.Reader

    clientConfig ClientConfig
    loadingLock  sync.Mutex

    // provided for testing
    icc InClusterConfig
}
{{< /highlight >}}


## ClientConfigLoadingRules.Load
{{< highlight go "linenos=inline" >}}
 // Load starts by running the MigrationRules and then
 // takes the loading rules and returns a Config object based on following rules.
 //   if the ExplicitPath, return the unmerged explicit file
 //   Otherwise, return a merged config based on the Precedence slice
 // A missing ExplicitPath file produces an error. Empty filenames or other missing files are ignored.
 // Read errors or files with non-deserializable content produce errors.
 // The first file to set a particular map key wins and map key's value is never changed.
 // BUT, if you set a struct value that is NOT contained inside of map, the value WILL be changed.
 // This results in some odd looking logic to merge in one direction, merge in the other, and then merge the two.
 // It also means that if two files specify a "red-user", only values from the first file's red-user are used.  Even
 // non-conflicting entries from the second file's "red-user" are discarded.
 // Relative paths inside of the .kubeconfig files are resolved against the .kubeconfig file's parent folder
 // and only absolute file paths are returned.
 // clientcmdapi "k8s.io/client-go/tools/clientcmd/api"
 func (rules *ClientConfigLoadingRules) Load() (*clientcmdapi.Config, error) {
     if err := rules.Migrate(); err != nil {
         return nil, err
     }

     errlist := []error{}
     missingList := []string{}

     kubeConfigFiles := []string{}

     // Make sure a file we were explicitly told to use exists
     if len(rules.ExplicitPath) > 0 {
         if _, err := os.Stat(rules.ExplicitPath); os.IsNotExist(err) {
             return nil, err
         }
         kubeConfigFiles = append(kubeConfigFiles, rules.ExplicitPath)

     } else {
         kubeConfigFiles = append(kubeConfigFiles, rules.Precedence...)
     }

     kubeconfigs := []*clientcmdapi.Config{}
     // read and cache the config files so that we only look at them once
     for _, filename := range kubeConfigFiles {
         if len(filename) == 0 {
             // no work to do
             continue
         }
         // 1. 从文件路径中加载kubeconfig
         config, err := LoadFromFile(filename)

         if os.IsNotExist(err) {
             // skip missing files
             // Add to the missing list to produce a warning
             missingList = append(missingList, filename)
             continue
         }

         if err != nil {
             errlist = append(errlist, fmt.Errorf("error loading config file \"%s\": %v", filename, err))
             continue
         }

         kubeconfigs = append(kubeconfigs, config)
     }

     if rules.WarnIfAllMissing && len(missingList) > 0 && len(kubeconfigs) == 0 {
         klog.Warningf("Config not found: %s", strings.Join(missingList, ", "))
     }

    // first merge all of our maps
    mapConfig := clientcmdapi.NewConfig()

    for _, kubeconfig := range kubeconfigs {
        mergo.MergeWithOverwrite(mapConfig, kubeconfig)
    }

    // merge all of the struct values in the reverse order so that priority is given correctly
    // errors are not added to the list the second time
    nonMapConfig := clientcmdapi.NewConfig()
    for i := len(kubeconfigs) - 1; i >= 0; i-- {
        kubeconfig := kubeconfigs[i]
        mergo.MergeWithOverwrite(nonMapConfig, kubeconfig)
    }

    // since values are overwritten, but maps values are not, we can merge the non-map config on top of the map config and
    // get the values we expect.
    config := clientcmdapi.NewConfig()
    mergo.MergeWithOverwrite(config, mapConfig)
    mergo.MergeWithOverwrite(config, nonMapConfig)
    if rules.ResolvePaths() {
        if err := ResolveLocalPaths(config); err != nil {
            errlist = append(errlist, err)
        }
    }
    return config, utilerrors.NewAggregate(errlist)
}
{{< /highlight >}}

## Load
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/tools/clientcmd/loader.go
// Load takes a byte slice and deserializes the contents into Config object.
// Encapsulates deserialization without assuming the source is a file.
func Load(data []byte) (*clientcmdapi.Config, error) {
    config := clientcmdapi.NewConfig()
    // if there's no data in a file, return the default object instead of failing (DecodeInto reject empty input)
    if len(data) == 0 {
        return config, nil
    }
    decoded, _, err := clientcmdlatest.Codec.Decode(data, &schema.GroupVersionKind{Version: clientcmdlatest.Version, Kind: "Config"}, config)
    if err != nil {
        return nil, err
    }
    return decoded.(*clientcmdapi.Config), nil
}
{{< /highlight >}}

## MergeWithOverwrite
{{< highlight go "linenos=inline" >}}
// MergeWithOverwrite will do the same as Merge except that non-empty dst attributes will be overridden by
// non-empty src attribute values.
// Deprecated: use Merge(…) with WithOverride
func MergeWithOverwrite(dst, src interface{}, opts ...func(*Config)) error {
    return merge(dst, src, append(opts, WithOverride)...)
}
{{< /highlight >}}

## merge
{{< highlight go "linenos=inline" >}}
 34 func merge(dst, src interface{}, opts ...func(*Config)) error {
 33     if dst != nil && reflect.ValueOf(dst).Kind() != reflect.Ptr {
 32         return ErrNonPointerArgument
 31     }
 30     var (
 29         vDst, vSrc reflect.Value
 28         err        error
 27     )
 26
 25     config := &Config{}
 24
 23     for _, opt := range opts {
 22         opt(config)
 21     }
 20
 19     if vDst, vSrc, err = resolveValues(dst, src); err != nil {
 18         return err
 17     }
 16     if !vDst.CanSet() {
 15         return fmt.Errorf("cannot set dst, needs reference")
 14     }
 13     if vDst.Type() != vSrc.Type() {
 12         return ErrDifferentArgumentsTypes
 11     }
 10     _, err = deepMerge(vDst, vSrc, make(map[uintptr]*visit), 0, config)
  9     return err
  8 }
{{< /highlight >}}

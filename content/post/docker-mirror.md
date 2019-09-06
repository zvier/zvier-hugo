---
title: "Docker Mirror服务搭建(编辑中)"
date: 2019-08-03T07:06:55+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---

# Registry作为仓库代理简介
Registry作为仓库代理使用时，可以用于镜像获取加速，同时起到备份的功能。不过工作在代理模式下的Registry只能用于镜像读取，首次获取某个镜像的请求，将会被透明地代理到远程原始仓库，然后缓存到代理仓库，待下一次有用户请求相同镜像时，便可以直接从缓存中获取。此时，该Registry也叫原始仓的Mirror。


# 搭建代理Registry
以下步骤可以搭建一个基于htttps的代理仓，仅供参考

## 制作证书
执行命令如下命令，生成服务器证书、服务器私钥等文件
{{< highlight bash "linenos=inline" >}}
domain=www.zvier.top
openssl req -nodes -subj "/C=CN/ST=Xian/L=Gaoxin/CN=${domain}" \ 
            -newkey rsa:4096 -keyout ${domain}.key -out ${domain}.csr
openssl x509 -req -days 3650 -in ${domain}.csr -signkey ${domain}.key \
            -out ${domain}.crt
{{< /highlight >}}

然后将服务器证书www.zvier.top.crt和服务器私钥www.zvier.top.key拷贝到/root/certs目录  


## 创建仓库存储目录
{{< highlight bash "linenos=inline" >}}
mkdir /opt/registry
{{< /highlight >}}

## 创建Registry容器
{{< highlight bash "linenos=inline" >}}
domain=www.zvier.top
docker run -d -p 443:5000 --restart=always --name registry \
           -v /opt/registry:/var/lib/registry -v /root/certs:/certs \
           -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/${domain}.crt \
           -e REGISTRY_HTTP_TLS_KEY=/certs/${domain}.key registry:2
{{< /highlight >}}

## 调整Registry配置
{{< highlight bash "linenos=inline" >}}
docker exec -it registry /bin/sh
echo "proxy:" >> /etc/docker/registry/config.yml
echo "    remoteurl: https://index.docker.io" >> /etc/docker/registry/config.yml
{{< /highlight >}}

{{< highlight bash "linenos=inline" >}}
docker restart registry
{{< /highlight >}}

# 客户端
## 让docker信任证书
{{< highlight bash "linenos=inline" >}}
domain=www.zvier.top
mkdir -p /etc/docker/certs.d/${domain}
openssl s_client -connect ${domain}:443 -showcerts </dev/null 2>/dev/null \
        | openssl x509 -outform PEM \
        | sudo tee /etc/docker/certs.d/${domain}/ca.crt
{{< /highlight >}}

这一步其实就是把证书拷到 /etc/docker/certs.d/${domain}/ 目录下。

## 让操作系统信任证书
{{< highlight bash "linenos=inline" >}}
domain=www.zvier.top
openssl s_client -connect ${domain}:443 -showcerts </dev/null 2>/dev/null \
        | openssl x509 -outform PEM \
        | sudo tee /etc/pki/ca-trust/source/anchors/$DOMAIN_NAME.crt
update-ca-trust
{{< /highlight >}}

## 配置Docker Mirrors
{{< highlight bash "linenos=inline" >}}
vim /etc/docker/daemon.json
{{< /highlight >}}

{{< highlight bash "linenos=inline" >}}
{
  "registry-mirrors": ["https://www.zvier.top"]
}
{{< /highlight >}}

## 重启 Docker
{{< highlight bash "linenos=inline" >}}
systemctl restart docker
{{< /highlight >}}

## 测试验证
拉取一个mysql镜像并计时

{{< highlight bash "linenos=inline" >}}
time docker pull mysql
Using default tag: latest
latest: Pulling from library/mysql
0a4690c5d889: Pull complete
98aa2fc6cbeb: Pull complete
......
Digest: sha256:b1b2c176a45f4ff6875d0d7461fe1fdd8606deb015b38b69545f72de97714efd
Status: Downloaded newer image for mysql:latest

real	1m17.641s
user	0m0.102s
sys	0m0.050s
{{< /highlight >}}

由于是第一次垃取，服务器上没缓存，会慢一些，耗时1min+。 拉取完毕后，查看下Registry镜像目录缓存内容，如下
{{< highlight bash "linenos=inline" >}}
ls /opt/registry/docker/registry/v2/repositories/library/
busybox  centos  hello-world  mysql  redis  traefik  ubuntu
{{< /highlight >}}

把客户端的 images 删除，然后重新拉取一下，并计时。
{{< highlight bash "linenos=inline" >}}
docker rmi mysql
Untagged: mysql:latest
Untagged: mysql@sha256:b1b2c176a45f4ff6875d0d7461fe1fdd8606deb015b38b69545f72de97714efd
Deleted: sha256:2151acc128817cb6ad0eecd897e31474d860217f56787d7f8ae645093022d096
Deleted: sha256:3f22fec6a0d06cb6ba8100a1f8a9ba11ac8a8f300d57fa79f78bf1dfa773f656
...
{{< /highlight >}}
客户端镜像虽删除，但Registry缓存镜像还在  
{{< highlight bash "linenos=inline" >}}
ls /opt/registry/docker/registry/v2/repositories/library/
busybox  centos  hello-world  mysql  redis  traefik  ubuntu
{{< /highlight >}}

{{< highlight bash "linenos=inline" >}}
time docker pull mysql
Using default tag: latest
latest: Pulling from library/mysql
0a4690c5d889: Pull complete
98aa2fc6cbeb: Pull complete
......
Digest: sha256:b1b2c176a45f4ff6875d0d7461fe1fdd8606deb015b38b69545f72de97714efd
Status: Downloaded newer image for mysql:latest

real	0m15.431s
user	0m0.055s
sys	0m0.027s
{{< /highlight >}}

# 相关源码分析
以下从源码角度分析了整个镜像获取相关的流程，仅供参考

## 镜像名组成
{{< highlight bash "linenos=inline" >}}
docker images
REPOSITORY                                                                          TAG                 IMAGE ID            CREATED             SIZE
traefik                                                                             latest              fb5ce07475c6        4 months ago        71MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64            v1.12.0             ab60b017e34f        10 months ago       194MB
quay.io/coreos/flannel                                                              v0.10.0-amd64       f0fad859c909        18 months ago       44.6MB
{{< /highlight >}}

完整的镜像表示包含Registry，Repository, Tag等几个部分，Tag和ImageID相关联，Tag和IMAGE ID可以多对一。

### Registry
quay.io/coreos和registry.cn-hangzhou.aliyuncs.com/google_containers就是Registry部分，用于存储镜像数据，提供垃取和上传镜像等功能

### Repository
quay.io/coreos/flannel或registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver称为一个Repository，一个Registry可以包含一个或多个Repository

### ImageID
Repository包含一个或多个Image，Image用GUID表示，有一个或多个Tag与之关联

## docker daemon启动加载daemon.json
docker daemon在启动时，new了一个DaemonCommand，其中包括调用getDefaultDaemonConfigFile以及从命令行参数中获取config-file文件，默认即/etc/docker/daemon.json
{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/cmd/dockerd/docker.go
func main() {
    ...
    cmd, err := newDaemonCommand()
    ...
}

func newDaemonCommand() (*cobra.Command, error) {
    opts := newDaemonOptions(config.New())
    cmd := &cobra.Command{
        Use:           "dockerd [OPTIONS]",
        ...
        RunE: func(cmd *cobra.Command, args []string) error {
            opts.flags = cmd.Flags()
            return runDaemon(opts)
        },
    ...
    flags := cmd.Flags()
    ...
    defaultDaemonConfigFile, err := getDefaultDaemonConfigFile()
    ...
    flags.StringVar(&opts.configFile, "config-file", defaultDaemonConfigFile, "Daemon configuration file")
    opts.InstallFlags(flags)
    if err := installConfigFlags(opts.daemonConfig, flags); err != nil {
        return nil, err
    }
    ...
    return cmd, nil
}
{{< /highlight >}}

{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/cmd/dockerd/daemon_unix.go
func getDefaultDaemonConfigFile() (string, error) {
    ...
    dir, err := getDefaultDaemonConfigDir()
    ...
    return filepath.Join(dir, "daemon.json"), nil
}

func getDefaultDaemonConfigDir() (string, error)
    if !honorXDG {
        return "/etc/docker", nil
    }
    ...
}
{{< /highlight >}}

安装配置项，在其中安装RegistryService相关的配置项时，就包括registry-mirror相关的内容
{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/cmd/dockerd/config_unix.go
// installConfigFlags adds flags to the pflag.FlagSet to configure the daemon
func installConfigFlags(conf *config.Config, flags *pflag.FlagSet) error {
    // First handle install flags which are consistent cross-platform
    if err := installCommonConfigFlags(conf, flags); err != nil {
        return err
    }
    ...
}

{{< /highlight >}}
{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/cmd/dockerd/config.go
// installCommonConfigFlags adds flags to the pflag.FlagSet to configure the daemon
func installCommonConfigFlags(conf *config.Config, flags *pflag.FlagSet) error {
    ...
    installRegistryServiceFlags(&conf.ServiceOptions, flags)
    ...
}


func installRegistryServiceFlags(options *registry.ServiceOptions, flags *pflag.FlagSet) {
    ana := opts.NewNamedListOptsRef("allow-nondistributable-artifacts", &options.AllowNondistributableArtifacts, registry.ValidateIndexName)
    mirrors := opts.NewNamedListOptsRef("registry-mirrors", &options.Mirrors, registry.ValidateMirror)
    insecureRegistries := opts.NewNamedListOptsRef("insecure-registries", &options.InsecureRegistries, registry.ValidateIndexName)

    flags.Var(ana, "allow-nondistributable-artifacts", "Allow push of nondistributable artifacts to registry")
    flags.Var(mirrors, "registry-mirror", "Preferred Docker registry mirror")
    flags.Var(insecureRegistries, "insecure-registry", "Enable insecure registry communication")
}
{{< /highlight >}}

## docker daemon启动
在runDaemon中，包括初始化一个DaemonCli，然后这个DaemonCli会带着daemonOptions启动，DaemonCli在start时，会New一个Daemon，初始化registryService，设置和初始化api请求路由器等。  

{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/cmd/dockerd/docker_unix.go
 func runDaemon(opts *daemonOptions) error {
     daemonCli := NewDaemonCli()
     return daemonCli.start(opts)
 }
{{< /highlight >}}

{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/cmd/dockerd/daemon.go
// DaemonCli represents the daemon CLI.
type DaemonCli struct {
    *config.Config
    configFile *string
    flags      *pflag.FlagSet

    api             *apiserver.Server
    d               *daemon.Daemon
    authzMiddleware *authorization.Middleware // authzMiddleware enables to dynamically reload the authorization plugins
}

// NewDaemonCli returns a daemon CLI
func NewDaemonCli() *DaemonCli {
    return &DaemonCli{}
}

type routerOptions struct {
    sessionManager *session.Manager
    buildBackend   *buildbackend.Backend
    buildCache     *fscache.FSCache // legacy
    features       *map[string]bool
    buildkit       *buildkit.Builder
    daemon         *daemon.Daemon
    api            *apiserver.Server
    cluster        *cluster.Cluster
}

func (cli *DaemonCli) start(opts *daemonOptions) (err error) {
    ...
    d, err := daemon.NewDaemon(ctx, cli.Config, pluginStore)
    ...
    routerOptions, err := newRouterOptions(cli.Config, d)
    ...
    initRouter(routerOptions)
    ...
}

// NewDaemon sets up everything for the daemon to be able to service
// requests from the webserver.
func NewDaemon(ctx context.Context, config *config.Config, pluginStore *plugin.Store) (daemon *Daemon, err error) {
    setDefaultMtu(config)
    registryService, err := registry.NewService(config.ServiceOptions)
    ...
    // TODO: imageStore, distributionMetadataStore, and ReferenceStore are only
    // used above to run migration. They could be initialized in ImageService
    // if migration is called from daemon/images. layerStore might move as well.
    d.imageService = images.NewImageService(images.ImageServiceConfig{
        ContainerStore:            d.containers,
        DistributionMetadataStore: distributionMetadataStore,
        EventsService:             d.EventsService,
        ImageStore:                imageStore,
        LayerStores:               layerStores,
        MaxConcurrentDownloads:    *config.MaxConcurrentDownloads,
        MaxConcurrentUploads:      *config.MaxConcurrentUploads,
        ReferenceStore:            rs,
        RegistryService:           registryService,
        TrustKey:                  trustKey,
    })
    ...
}

func initRouter(opts routerOptions) {
    ...
    routers := []router.Router{
        ...
        image.NewRouter(opts.daemon.ImageService()),
        ...
     }
    ...
{{< /highlight >}}

{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/registry/service.go
// DefaultService is a registry service. It tracks configuration data such as a list
// of mirrors.
type DefaultService struct {
    config *serviceConfig
    mu     sync.Mutex
}

// NewService returns a new instance of DefaultService ready to be
// installed into an engine.
func NewService(options ServiceOptions) (*DefaultService, error) {
    config, err := newServiceConfig(options)

    return &DefaultService{config: config}, err
}
{{< /highlight >}}

## 查询endpoints
{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/registry/service.go
// LookupPullEndpoints creates a list of endpoints to try to pull from, in order of preference.
// It gives preference to v2 endpoints over v1, mirrors over the actual
// registry, and HTTPS over plain HTTP.
func (s *DefaultService) LookupPullEndpoints(hostname string) (endpoints []APIEndpoint, err error) {
    s.mu.Lock()
    defer s.mu.Unlock()

    return s.lookupEndpoints(hostname)
}
func (s *DefaultService) lookupEndpoints(hostname string) (endpoints []APIEndpoint, err error) {
    return s.lookupV2Endpoints(hostname)
}
{{< /highlight >}}

这里DefaultNamespace = "docker.io"，IndexHostname = "index.docker.io"，只有当Registry为这二者之一时，才会去加载mirrors中的endpoint，而registry的url置放在endpoints数组末尾
{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/registry/service_v2.go
func (s *DefaultService) lookupV2Endpoints(hostname string) (endpoints []APIEndpoint, err error) {
    tlsConfig := tlsconfig.ServerDefault()
    if hostname == DefaultNamespace || hostname == IndexHostname {
        // v2 mirrors
        for _, mirror := range s.config.Mirrors {
            if !strings.HasPrefix(mirror, "http://") && !strings.HasPrefix(mirror, "https://") {
                mirror = "https://" + mirror
            }
            mirrorURL, err := url.Parse(mirror)
            if err != nil {
                return nil, err
            }
            mirrorTLSConfig, err := s.tlsConfigForMirror(mirrorURL)
            if err != nil {
                return nil, err
            }
            endpoints = append(endpoints, APIEndpoint{
                URL: mirrorURL,
                // guess mirrors are v2
                Version:      APIVersion2,
                Mirror:       true,
                TrimHostname: true,
                TLSConfig:    mirrorTLSConfig,
            })
        }
        // v2 registry
        endpoints = append(endpoints, APIEndpoint{
            URL:          DefaultV2Registry,
            Version:      APIVersion2,
            Official:     true,
            TrimHostname: true,
            TLSConfig:    tlsConfig,
        })

        return endpoints, nil
    }
{{< /highlight >}}

{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/registry/config.go
var (
    // DefaultNamespace is the default namespace
    DefaultNamespace = "docker.io"
    // DefaultRegistryVersionHeader is the name of the default HTTP header
    // that carries Registry version info
    DefaultRegistryVersionHeader = "Docker-Distribution-Api-Version"

    // IndexHostname is the index hostname
    IndexHostname = "index.docker.io"
    // IndexServer is used for user auth and image search
    IndexServer = "https://" + IndexHostname + "/v1/"
    // IndexName is the name of the index
    IndexName = "docker.io"

    // DefaultV2Registry is the URI of the default v2 registry
    DefaultV2Registry = &url.URL{
        Scheme: "https",
        Host:   "registry-1.docker.io",
    }
)
// ServiceOptions holds command line options.
type ServiceOptions struct {
    AllowNondistributableArtifacts []string `json:"allow-nondistributable-artifacts,omitempty"`
    Mirrors                        []string `json:"registry-mirrors,omitempty"`
    InsecureRegistries             []string `json:"insecure-registries,omitempty"`
}

// serviceConfig holds daemon configuration for the registry service.
type serviceConfig struct {
    registrytypes.ServiceConfig
}

// newServiceConfig returns a new instance of ServiceConfig
func newServiceConfig(options ServiceOptions) (*serviceConfig, error) {
    config := &serviceConfig{
        ServiceConfig: registrytypes.ServiceConfig{
            InsecureRegistryCIDRs: make([]*registrytypes.NetIPNet, 0),
            IndexConfigs:          make(map[string]*registrytypes.IndexInfo),
            // Hack: Bypass setting the mirrors to IndexConfigs since they are going away
            // and Mirrors are only for the official registry anyways.
        },
    }
    ...
    if err := config.LoadMirrors(options.Mirrors); err != nil {
        return nil, err
    }
    if err := config.LoadInsecureRegistries(options.InsecureRegistries); err != nil {
        return nil, err
    }

    return config, nil
}
// LoadMirrors loads mirrors to config, after removing duplicates.
// Returns an error if mirrors contains an invalid mirror.
func (config *serviceConfig) LoadMirrors(mirrors []string) error {
    mMap := map[string]struct{}{}
    unique := []string{}

    for _, mirror := range mirrors {
        m, err := ValidateMirror(mirror)
        if err != nil {
            return err
        }
        if _, exist := mMap[m]; !exist {
            mMap[m] = struct{}{}
            unique = append(unique, m)
        }
    }

    config.Mirrors = unique

    // Configure public registry since mirrors may have changed.
    config.IndexConfigs[IndexName] = &registrytypes.IndexInfo{
        Name:     IndexName,
        Mirrors:  config.Mirrors,
        Secure:   true,
        Official: true,
    }

    return nil
}
{{< /highlight >}}

{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/api/types/registry/registry.go
type ServiceConfig struct {
    AllowNondistributableArtifactsCIDRs     []*NetIPNet
    AllowNondistributableArtifactsHostnames []string
    InsecureRegistryCIDRs                   []*NetIPNet           `json:"InsecureRegistryCIDRs"`
    IndexConfigs                            map[string]*IndexInfo `json:"IndexConfigs"`
    Mirrors                                 []string
}

type IndexInfo struct {
    // Name is the name of the registry, such as "docker.io"
    Name string
    // Mirrors is a list of mirrors, expressed as URIs
    Mirrors []string
    // Secure is set to false if the registry is part of the list of
    // insecure registries. Insecure registries accept HTTP and/or accept
    // HTTPS with certificates from unknown CAs.
    Secure bool
    // Official indicates whether this is an official registry
    Official bool
}
{{< /highlight >}}

/images/create api相关的请求路由
{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/api/server/router/image/image.go
// imageRouter is a router to talk with the image controller
type imageRouter struct {
    backend Backend
    routes  []router.Route
}

// NewRouter initializes a new image router
func NewRouter(backend Backend) router.Router {
    r := &imageRouter{backend: backend}
    r.initRoutes()
    return r
}

// initRoutes initializes the routes in the image router
func (r *imageRouter) initRoutes() {
    ...
    router.NewPostRoute("/images/create", r.postImagesCreate),
    ...
}
{{< /highlight >}}

{{< highlight golang "linenos=inline" >}}
// localRoute defines an individual API route to connect
// with the docker daemon. It implements Route.
type localRoute struct {
    method  string
    path    string
    handler httputils.APIFunc
}
// NewRoute initializes a new local route for the router.
func NewRoute(method, path string, handler httputils.APIFunc, opts ...RouteWrapper) Route {
    var r Route = localRoute{method, path, handler}
    for _, o := range opts {
        r = o(r)
    }
    return r
}
 // NewPostRoute initializes a new route with the http method POST.
 func NewPostRoute(path string, handler httputils.APIFunc, opts ...RouteWrapper) Route {
     return NewRoute("POST", path, handler, opts...)
 }
 
{{< /highlight >}}

{{< highlight golang "linenos=inline" >}}
// github.com/docker/dockerapi/server/router/image/image_routes.go 
// Creates an image from Pull or from Import
func (s *imageRouter) postImagesCreate(ctx context.Context, w http.ResponseWriter, r *http.Request, vars map[string]string) error {
    ...
    var (
       image    = r.Form.Get("fromImage")
       repo     = r.Form.Get("repo")
       ...
    )
    if image != "" { 
        ...
        err = s.backend.PullImage(ctx, image, tag, platform, metaHeaders, authConfig, output)
        ...       
    }
}
{{< /highlight >}}

{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/registry/types.go
// RepositoryInfo describes a repository
type RepositoryInfo struct {
    Name reference.Named
    // Index points to registry information
    Index *registrytypes.IndexInfo
    // Official indicates whether the repository is considered official.
    // If the registry is official, and the normalized name does not
    // contain a '/' (e.g. "foo"), then it is considered an official repo.
    Official bool
    // Class represents the class of the repository, such as "plugin"
    // or "image".
    Class string
}
{{< /highlight >}}

{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/daemon/images/image_pull.go
 // PullImage initiates a pull operation. image is the repository name to pull, and
 // tag may be either empty, or indicate a specific tag to pull.
 func (i *ImageService) PullImage(ctx context.Context, image, tag string, platform *specs.Platform, metaHeaders map[string][]string, authConfig *types.AuthConfig, outStream io.Writer) error {
     ...
     err = i.pullImageWithReference(ctx, ref, platform, metaHeaders, authConfig, outStream)
     ...

func (i *ImageService) pullImageWithReference(ctx context.Context, ref reference.Named, platform *specs.Platform, metaHeaders map[string][]string, authConfig *types.Auth    Config, outStream io.Writer) error {
    ...
    imagePullConfig := &distribution.ImagePullConfig{
        Config: distribution.Config{
            MetaHeaders:      metaHeaders,
            AuthConfig:       authConfig,
            ProgressOutput:   progress.ChanOutput(progressChan),
            RegistryService:  i.registryService,
            ImageEventLogger: i.LogImageEvent,
            MetadataStore:    i.distributionMetadataStore,
            ImageStore:       distribution.NewImageConfigStoreFromStore(i.imageStore),
            ReferenceStore:   i.referenceStore,
        },
        DownloadManager: i.downloadManager,
        Schema2Types:    distribution.ImageTypes,
        Platform:        platform,
    }

    err := distribution.Pull(ctx, ref, imagePullConfig)
    ...
}
{{< /highlight >}}

## 发起Pull请求
在真正Pull请求时，会去查询所有可获取镜像的endpoints，然后逐遍历这些endpoint，创建Puller，尝试拉取镜像，如果失败，则继续下一个。
{{< highlight golang "linenos=inline" >}}
// github.com/docker/docker/distribution/pull.go
// Pull initiates a pull operation. image is the repository name to pull, and
// tag may be either empty, or indicate a specific tag to pull.
func Pull(ctx context.Context, ref reference.Named, imagePullConfig *ImagePullConfig) error {
    // Resolve the Repository name from fqn to RepositoryInfo
    repoInfo, err := imagePullConfig.RegistryService.ResolveRepository(ref)
    ...
    endpoints, err := imagePullConfig.RegistryService.LookupPullEndpoints(reference.Domain(repoInfo.Name))
    ...
    for _, endpoint := range endpoints {
        ...
        logrus.Debugf("Trying to pull %s from %s %s", reference.FamiliarName(repoInfo.Name), endpoint.URL, endpoint.Version)
        puller, err := newPuller(endpoint, repoInfo, imagePullConfig)
        ...
        if err := puller.Pull(ctx, ref, imagePullConfig.Platform); err != nil {
            // Was this pull cancelled? If so, don't try to fall
            // back.
            fallback := false
            select {
            case <-ctx.Done():
            default:
                if fallbackErr, ok := err.(fallbackError); ok {
                    fallback = true
                    confirmedV2 = confirmedV2 || fallbackErr.confirmedV2
                    if fallbackErr.transportOK && endpoint.URL.Scheme == "https" {
                        confirmedTLSRegistries[endpoint.URL.Host] = struct{}{}
                    }
                    err = fallbackErr.err
                }
            }
            if fallback {
                if _, ok := err.(ErrNoSupport); !ok {
                    // Because we found an error that's not ErrNoSupport, discard all subsequent ErrNoSupport errors.
                    discardNoSupportErrors = true
                    // append subsequent errors
                    lastErr = err
                } else if !discardNoSupportErrors {
                    // Save the ErrNoSupport error, because it's either the first error or all encountered errors
                    // were also ErrNoSupport errors.
                    // append subsequent errors
                    lastErr = err
                }
                logrus.Infof("Attempting next endpoint for pull after error: %v", err)
                continue
            }
            logrus.Errorf("Not continuing with pull after error: %v", err)
            return TranslatePullError(err, ref)
        }

        imagePullConfig.ImageEventLogger(reference.FamiliarString(ref), reference.FamiliarName(repoInfo.Name), "pull")
        return nil
     }

     if lastErr == nil {
         lastErr = fmt.Errorf("no endpoints found for %s", reference.FamiliarString(ref))
     }

     return TranslatePullError(lastErr, ref)
}

{{< /highlight >}}

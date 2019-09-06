---
title: "CNI 容器网络接口"
date: 2019-07-06T07:29:20+08:00
draft: true
categories: ["云原生"]
tags: ["k8s", "docker"]
---

# 什么是CNI
k8s中，为了实现POD间通信，需要虚拟出一个网络通信平面，当前构建这个虚拟网络平面的方案五花八门，常见的有flannel，calico，openvswitch，weave，ipvlan等，如果不形成一个规范统一的标准，会造成模块之间大量的适配工作。

CNI(Container Network Interface)就是这样一个容器网络接口规范，或者说是一个容器网络协议，作为连接容器管理系统和网络插件的接口，它约定了在容器创建或删除时的方法和参数等。任何满足这个规范的实现，都可以很好地支持上层容器编排系统的工作，k8s、mesos等可以直接通过调用这组约定的接口对容器网络进行操作和配置，而无需关注插件的实现细节。

当然，CNI中还包含镜像相关的接口定义，因为镜像的生命周期和容器运行时彼此隔离，因此需要定义两个服务: RuntimeServiceServer和ImageServiceServer。

## RuntimeServiceServer
{{< highlight go "linenos=inline" >}}
k8s.io/cri-api/pkg/apis/runtime/v1alpha2/api.pb.go
// Server API for RuntimeService service
type RuntimeServiceServer interface {
    // Version returns the runtime name, runtime version, and runtime API version.
    Version(context.Context, *VersionRequest) (*VersionResponse, error)
    // RunPodSandbox creates and starts a pod-level sandbox. Runtimes must ensure
    // the sandbox is in the ready state on success.
    RunPodSandbox(context.Context, *RunPodSandboxRequest) (*RunPodSandboxResponse, error)
    // StopPodSandbox stops any running process that is part of the sandbox and
    // reclaims network resources (e.g., IP addresses) allocated to the sandbox.
    // If there are any running containers in the sandbox, they must be forcibly
    // terminated.
    // This call is idempotent, and must not return an error if all relevant
    // resources have already been reclaimed. kubelet will call StopPodSandbox
    // at least once before calling RemovePodSandbox. It will also attempt to
    // reclaim resources eagerly, as soon as a sandbox is not needed. Hence,
    // multiple StopPodSandbox calls are expected.
    StopPodSandbox(context.Context, *StopPodSandboxRequest) (*StopPodSandboxResponse, error)
    // RemovePodSandbox removes the sandbox. If there are any running containers
    // in the sandbox, they must be forcibly terminated and removed.
    // This call is idempotent, and must not return an error if the sandbox has
    // already been removed.
    RemovePodSandbox(context.Context, *RemovePodSandboxRequest) (*RemovePodSandboxResponse, error)
    // PodSandboxStatus returns the status of the PodSandbox. If the PodSandbox is not
    // present, returns an error.
    PodSandboxStatus(context.Context, *PodSandboxStatusRequest) (*PodSandboxStatusResponse, error)
    // ListPodSandbox returns a list of PodSandboxes.
    ListPodSandbox(context.Context, *ListPodSandboxRequest) (*ListPodSandboxResponse, error)
    // CreateContainer creates a new container in specified PodSandbox
    CreateContainer(context.Context, *CreateContainerRequest) (*CreateContainerResponse, error)
    // StartContainer starts the container.
    StartContainer(context.Context, *StartContainerRequest) (*StartContainerResponse, error)
    // StopContainer stops a running container with a grace period (i.e., timeout).
    // This call is idempotent, and must not return an error if the container has
    // already been stopped.
    // TODO: what must the runtime do after the grace period is reached?
    StopContainer(context.Context, *StopContainerRequest) (*StopContainerResponse, error)
    // RemoveContainer removes the container. If the container is running, the
    // container must be forcibly removed.
    // This call is idempotent, and must not return an error if the container has
    // already been removed.
    RemoveContainer(context.Context, *RemoveContainerRequest) (*RemoveContainerResponse, error)
    // ListContainers lists all containers by filters.
    ListContainers(context.Context, *ListContainersRequest) (*ListContainersResponse, error)
    // ContainerStatus returns status of the container. If the container is not
    // present, returns an error.
    ContainerStatus(context.Context, *ContainerStatusRequest) (*ContainerStatusResponse, error)
    // UpdateContainerResources updates ContainerConfig of the container.
    UpdateContainerResources(context.Context, *UpdateContainerResourcesRequest) (*UpdateContainerResourcesResponse, error)
    // ReopenContainerLog asks runtime to reopen the stdout/stderr log file
    // for the container. This is often called after the log file has been
    // rotated. If the container is not running, container runtime can choose
    // to either create a new log file and return nil, or return an error.
    // Once it returns error, new container log file MUST NOT be created.
    ReopenContainerLog(context.Context, *ReopenContainerLogRequest) (*ReopenContainerLogResponse, error)
    // ExecSync runs a command in a container synchronously.
    ExecSync(context.Context, *ExecSyncRequest) (*ExecSyncResponse, error)
    // Exec prepares a streaming endpoint to execute a command in the container.
    Exec(context.Context, *ExecRequest) (*ExecResponse, error)
    // Attach prepares a streaming endpoint to attach to a running container.
    Attach(context.Context, *AttachRequest) (*AttachResponse, error)
    // PortForward prepares a streaming endpoint to forward ports from a PodSandbox.
    PortForward(context.Context, *PortForwardRequest) (*PortForwardResponse, error)
    // ContainerStats returns stats of the container. If the container does not
    // exist, the call returns an error.
    ContainerStats(context.Context, *ContainerStatsRequest) (*ContainerStatsResponse, error)
    // ListContainerStats returns stats of all running containers.
    ListContainerStats(context.Context, *ListContainerStatsRequest) (*ListContainerStatsResponse, error)
    // UpdateRuntimeConfig updates the runtime configuration based on the given request.
    UpdateRuntimeConfig(context.Context, *UpdateRuntimeConfigRequest) (*UpdateRuntimeConfigResponse, error)
    // Status returns the status of the runtime.
    Status(context.Context, *StatusRequest) (*StatusResponse, error)
 }
{{< /highlight >}}

## ImageServiceServer
   35 // Server API for ImageService service
   34
   33 type ImageServiceServer interface {
   32     // ListImages lists existing images.
   31     ListImages(context.Context, *ListImagesRequest) (*ListImagesResponse, error)
   30     // ImageStatus returns the status of the image. If the image is not
   29     // present, returns a response with ImageStatusResponse.Image set to
   28     // nil.
   27     ImageStatus(context.Context, *ImageStatusRequest) (*ImageStatusResponse, error)
   26     // PullImage pulls an image with authentication config.
   25     PullImage(context.Context, *PullImageRequest) (*PullImageResponse, error)
   24     // RemoveImage removes the image.
   23     // This call is idempotent, and must not return an error if the image has
   22     // already been removed.
   21     RemoveImage(context.Context, *RemoveImageRequest) (*RemoveImageResponse, error)
   20     // ImageFSInfo returns information of the filesystem that is used to store images.
   19     ImageFsInfo(context.Context, *ImageFsInfoRequest) (*ImageFsInfoResponse, error)
   18 }


# CNI接口详解
## 接口定义
CNI操作的对象就是容器网络或网络列表，包括添加，检查，删表，获取，校验等方法，下面我们看以下相关的数据结构

### CNI
CNI是一个interface，约定了CNI插件的实现需要提供哪些方法(功能)   
{{< highlight go "linenos=inline" >}}
// github.com/containernetworking/cni/libcni/api.go
type CNI interface {
    AddNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)
    CheckNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) error
    DelNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) error
    GetNetworkListCachedResult(net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)

    AddNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) (types.Result, error)
    CheckNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) error
    DelNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) error
    GetNetworkCachedResult(net *NetworkConfig, rt *RuntimeConf) (types.Result, error)

    ValidateNetworkList(ctx context.Context, net *NetworkConfigList) ([]string, error)
    ValidateNetwork(ctx context.Context, net *NetworkConfig) ([]string, error)
}
{{< /highlight >}}

通过CNI interface可知，上层容器管理系统在调用底层网络插件时，要传递CNI配置和容器运行时配置，这些信息通过GRPC+JSON的方式在两个模块间交互，当然这里还涉及到一个dockershim的垫片，对容器网络的具体操作都由CNI插件来实现。
![cni-plugin-work-flow](/img/k8s/cni-plugin-work-flow.png)

## 接口参数
在上面接口定义的基础上，我们进一步分析下接口参数对应的实际内容。CNI插件是一个实现了CNI容器网络接口规范的二进制文件，从二进制插件的角度来看，插件的参数来自环境变量和标准输入，当容器管理系统发起GRPC+JSON的请求时，中间正是由类似dockershim的一个垫片，将调用信息转换成二进制文件的输入，调用信息有如下

### NetworkConfig或NetworkConfigList
网络配置: 网段，网关，DNS以及额外的信息等，NetworkConfig对应一份CNI网络配置，Bytes用于方便不定字段的扩充  
{{< highlight go "linenos=inline" >}}
type NetworkConfig struct {
    Network *types.NetConf
    Bytes   []byte
}
{{< /highlight >}}

NetworkConfigList对应着多份CNI配置，/etc/cni/net.d/目录下的.conflist文件内容就是一个NetworkConfigList      
{{< highlight go "linenos=inline" >}}
type NetworkConfigList struct {
    Name         string
    CNIVersion   string
    DisableCheck bool
    Plugins      []*NetworkConfig
    Bytes        []byte
}
{{< /highlight >}}

### RuntimeConf
容器管理系统在调用CNI插件时的容器运行时配置，比如ns文件，容器id等  
{{< highlight go "linenos=inline" >}}
// github.com/containernetworking/cni/libcni/api.go
// A RuntimeConf holds the arguments to one invocation of a CNI plugin
// excepting the network configuration, with the nested exception that
// the `runtimeConfig` from the network configuration is included here.
type RuntimeConf struct {
    ContainerID string
    NetNS       string
    IfName      string
    Args        [][2]string
    // A dictionary of capability-specific data passed by the runtime
    // to plugins as top-level keys in the 'runtimeConfig' dictionary
    // of the plugin's stdin data.  libcni will ensure that only keys
    // in this map which match the capabilities of the plugin are passed
    // to the plugin
    CapabilityArgs map[string]interface{}

    // DEPRECATED. Will be removed in a future release.
    CacheDir string
}
{{< /highlight >}}

# 执行流程
当网络配置和容器运行时配置都准备好后，再加上CNI实体本身的信息，比如插件位，插件名称等，就可以执行容器网络相关的操作了。举例看下AddNetwork的执行流程  
{{< highlight go "linenos=inline" >}}
// github.com/containernetworking/cni/libcni/api.go
// AddNetwork executes the plugin with the ADD command
func (c *CNIConfig) AddNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) (types.Result, error) {
    result, err := c.addNetwork(ctx, net.Network.Name, net.Network.CNIVersion, net, nil, rt)
    if err != nil {
        return nil, err
    }

    if err = c.setCachedResult(result, net.Network.Name, rt); err != nil {
        return nil, fmt.Errorf("failed to set network %q cached result: %v", net.Network.Name, err)
    }

    return result, nil
}
func (c *CNIConfig) addNetwork(ctx context.Context, name, cniVersion string, net *NetworkConfig, prevResult types.Result, rt *RuntimeConf) (types.Result, error) {
    c.ensureExec()
    pluginPath, err := c.exec.FindInPath(net.Network.Type, c.Path)
    if err != nil {
        return nil, err
    }

    newConf, err := buildOneConfig(name, cniVersion, net, prevResult, rt)
    if err != nil {
        return nil, err
    }

    return invoke.ExecPluginWithResult(ctx, pluginPath, newConf.Bytes, c.args("ADD", rt), c.exec)
}
{{< /highlight >}}
可见创建容器网络主要是根据解析到网络配置和容器运行时配置信息，查到到相应的二进制文件，并执行之    

{{< highlight go "linenos=inline" >}}
func buildOneConfig(name, cniVersion string, orig *NetworkConfig, prevResult types.Result, rt *RuntimeConf) (*NetworkConfig, error) {
    var err error

    inject := map[string]interface{}{
        "name":       name,
        "cniVersion": cniVersion,
    }
    // Add previous plugin result
    if prevResult != nil {
        inject["prevResult"] = prevResult
    }

    // Ensure every config uses the same name and version
    orig, err = InjectConf(orig, inject)
    if err != nil {
        return nil, err
    }

    return injectRuntimeConfig(orig, rt)
}

// github.com/containernetworking/cni/libcni/conf.go
func InjectConf(original *NetworkConfig, newValues map[string]interface{}) (*NetworkConfig, error) {
    config := make(map[string]interface{})
    err := json.Unmarshal(original.Bytes, &config)
    if err != nil {
        return nil, fmt.Errorf("unmarshal existing network bytes: %s", err)
    }

    for key, value := range newValues {
        if key == "" {
            return nil, fmt.Errorf("keys cannot be empty")
        }

        if value == nil {
            return nil, fmt.Errorf("key '%s' value must not be nil", key)
        }

        config[key] = value
    }

    newBytes, err := json.Marshal(config)
    if err != nil {
        return nil, err
    }

    return ConfFromBytes(newBytes)
}

func ConfFromBytes(bytes []byte) (*NetworkConfig, error) {
    conf := &NetworkConfig{Bytes: bytes}
    if err := json.Unmarshal(bytes, &conf.Network); err != nil {
        return nil, fmt.Errorf("error parsing configuration: %s", err)
    }
    if conf.Network.Type == "" {
        return nil, fmt.Errorf("error parsing configuration: missing 'type'")
    }
    return conf, nil
}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
20 func ExecPluginWithResult(ctx context.Context, pluginPath string, netconf []byte, args CNIArgs, exec Exec) (types.Result, error) {
 19     if exec == nil {
 18         exec = defaultExec
 17     }
 16
 15     stdoutBytes, err := exec.ExecPlugin(ctx, pluginPath, netconf, args.AsEnv())
 14     if err != nil {
 13         return nil, err
 12     }
 11
 10     // Plugin must return result in same version as specified in netconf
  9     versionDecoder := &version.ConfigDecoder{}
  8     confVersion, err := versionDecoder.Decode(netconf)
  7     if err != nil {
  6         return nil, err
  5     }
  4
  3     return version.NewResult(confVersion, stdoutBytes)
  2 }
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// github.com/containernetworking/cni/pkg/invoke/exec.go
// Exec is an interface encapsulates all operations that deal with finding
// and executing a CNI plugin. Tests may provide a fake implementation
// to avoid writing fake plugins to temporary directories during the test.
type Exec interface {
    ExecPlugin(ctx context.Context, pluginPath string, stdinData []byte, environ []string) ([]byte, error)
    FindInPath(plugin string, paths []string) (string, error)
    Decode(jsonBytes []byte) (version.PluginInfo, error)
}

// DefaultExec is an object that implements the Exec interface which looks
// for and executes plugins from disk.
type DefaultExec struct {
    *RawExec
    version.PluginDecoder
}
{{< /highlight >}}

来说，当我们把容器加入到网络时，以下信息将通过环境变量的方式进行传递  

* CNI_COMMAND: 要执行的操作，ADD(把容器加入到网络)，DEL(把容器从网络中删除)  
* CNI_CONTAINERID: 容器ID，比如ipam会把容器ID和分配的IP保存下来
* CNI_NETNS: 容器的network namespace文件，通过访问这个文件，可以在容器的网络中进行操作  
* CNI_IFNAME: 待配置的interface名，比如eth0   
* CNI_ARGS: 额外的参数，是由分号<code>;</code>分割的键值对，比如："key1=value1;key2=value2"
* CNI_PATH: CNI二进制查找的路径列表，多个路径用冒号<code>:</code>分隔

而网络信息主要通过标准输入注入，作为JSON字符串传递给插件，必要的参数有如下:  
* cniVersion: CNI标准的版本  
* name: 网络的名字  
* type: 网络插件的类型，即CNI可执行文件的名称  
* args: 额外的信息，是一个字典  
* ipMasq: 是否在主机上为该网络配置IP masquerade  
* ipam: IP分配相关的信息，类型为字典  
* dns: DNS相关的信息，类型为字典  

插件接受到这些信息后，执行相应的程序逻辑，然后把结果返回给调用者，返回结果一般包括如下信息:  
* IPs assigned to the interface：网络接口被分配的 ip，可以是 IPv4、IPv6 或者都有
* DNS 信息：包含 nameservers、domain、search domains 和其他选项的字典  

CNI 作为一个协议/标准，它有很强的扩展性和灵活性。如果用户对某个插件有额外的需求，可以通过输入中的 args 和环境变量 CNI_ARGS 传输，然后在插件中实现自定义的功能，这大大增加了它的扩展性；CNI 插件把 main 和 ipam 分开，用户可以自由组合它们，而且一个 CNI 插件也可以直接调用另外一个 CNI 插件，使用起来非常灵活。

如果要实现一个继承性的 CNI 插件也不复杂，可以编写自己的 CNI 插件，根据传入的配置调用 main 中已经有的插件，就能让用户自由选择容器的网络。


# CNI核心思想
容器运行时在创建容器时，先创建好网络命名空间，然后再按顺序调用CNI插件配置该网络命名空间，最后再将容器加入到网络命名空间，启动容器内的进程。  
网络配置形式如下，例：
{{< highlight json "linenos=inline" >}}
cat /etc/cni/net.d/10-flannel.conflist
{
  "name": "cbr0",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
{{< /highlight >}}

# CNI插件
CNI插件默认存放在/opt/cni/bin目录下
{{< highlight bash "linenos=inline" >}}
ls /opt/cni/bin
bridge	dhcp  flannel  host-local  ipvlan  loopback
macvlan  portmap  ptp  sample  tuning  vlan
{{< /highlight >}}

![cni-plugins](/img/k8s/cni-plugins.png)

这些插件又可分为如下几类：

## Main Plugin
Main类型的Plugin，负责给容器创建并配置网络接口，实现了某种特定功能的网络  

* bridge: 创建网桥，并添加主机和容器到该网桥  
* ipvlan: 在容器中添加一个ipvlan接口   
* loopback: 创建一个回环接口  
* macvlan: 建一个新的MAC地址，将所有的流量转发到容器   
* ptp: 创建veth对  
* vlan: 分配一个vlan设备    

### bridge
bridge是一种比较简单的CNI网络插件，和docker的默认网络模型类似，它首先在Host上创建一个网桥，然后再通过veth
pair将所有容器网络连接到网桥。如图:
![cni-bridge](/img/k8s/cni-bridge.png)

在bridge模式下，跨主机通信需要额外配置主机路由或使用overlay网络，下面是使用overlay构建虚拟网络以支持跨主机pod通信的拓扑图  
![cni-bridge](/img/k8s/cni-overlay.png)

### loopback
负责生成lo网卡，并配置127.0.0.1/8地址  

### macvlan
macvlan插件使用了macvlan技术，从某个物理网卡虚拟出多个虚拟网卡，它们有独立的ip和mac地址  

### ptp
通过veth pair在容器和主机之间建立通道


## IPAM Plugin
IPAM Plugin 本身并不提供具体的网络功能，主要用于给容器网络分配IP地址   

### dhcp
dhcp插件会从DHCP服务器中获取IP地址，然后分配给容器  

### host-local
host-local插件维护了一个基于本地文件数据库，它会把分配出去的IP保存在本地文件中  

## Meta Plugin
Meta Plugin本身也不提供具体的网络功能，它是通过调用其它插件实现  

### flannel
flannel插件会根据flannel的配置文件分配网络地址，然后调用bridge插件创建和配置容器接口  
tuning：调整现有接口的sysctl参数
portmap：一个基于iptables的portmapping插件。将端口从主机的地址空间映射到容器。

这些CNI插件负责将**<font color="red">网络接口</font>**插入到对应的容器网络命名空间内（例如，veth对的一端），并在主机上进行必要的改变（例如将veth的另一端连接到网桥）。然后将IP分配给接口，并通过调用适当的IPAM插件来设置与“IP地址管理”部分一致的路由。  


## 将容器添加到网络

## 从网络中删除容器

## IP分配

## IPAM插件

总之，CNI插件根据设定的游戏规则完成了容器网络的创建与配置。

# k8s POD创建流程分析
1. kubelet先创建pause容器，生成网络命名空间  
2. 调用网络CNI driver  
3. CNI driver根据配置调用具体的cni插件  
4. cni插件给pause容器配置网络  
5. pod中的其它容器共享pause容器网络  

# 源码分析

## 关键数据结构
### 容器运行时配置
在调用CNI插件时的容器运行时配置，不包含网络配置信息  

### CNI网络配置
下面是一份典型的CNI网络配置，这里cniVersion字段忽略了    
{{< highlight bash "linenos=inline" >}}
cat /etc/cni/net.d/cni-demo.conflist
{
    "cniVersion": "0.3.0",
    "name": "mynet",
    "plugins": [
        {
            "bridge": "mynet",
            "ipMasq": true,
            "ipam": {
                "routes": [
                    {
                        "dst": "0.0.0.0/0"
                    }
                ],
                "subnet": "10.244.10.0/24",
                "type": "host-local"
            },
            "isGateway": true,
            "type": "bridge"
        },
        {
            "capabilities": {
                "portMappings": true
            },
            "type": "portmap"
        }
    ]
}
{{< /highlight >}}

### NetConf
NetConf定义在github.com/containernetworking/cni/pkg/types/types.go文件中
{{< highlight go "linenos=inline" >}}
// NetConf describes a network.
type NetConf struct {
    CNIVersion string `json:"cniVersion,omitempty"`

    Name         string          `json:"name,omitempty"`
    Type         string          `json:"type,omitempty"`
    Capabilities map[string]bool `json:"capabilities,omitempty"`
    IPAM         IPAM            `json:"ipam,omitempty"`
    DNS          DNS             `json:"dns"`

    RawPrevResult map[string]interface{} `json:"prevResult,omitempty"`
    PrevResult    Result                 `json:"-"`
}
type IPAM struct {
    Type string `json:"type,omitempty"`
}

// NetConfList describes an ordered list of networks.
type NetConfList struct {
    CNIVersion string `json:"cniVersion,omitempty"`

    Name         string     `json:"name,omitempty"`
    DisableCheck bool       `json:"disableCheck,omitempty"`
    Plugins      []*NetConf `json:"plugins,omitempty"`
}
{{< /highlight >}}


### CNIConfig
CNIConfig实现了CNI  
{{< highlight go "linenos=inline" >}}
type CNIConfig struct {
    Path     []string
    exec     invoke.Exec
    cacheDir string
}
{{< /highlight >}}
# 参考
[CNI - Container Network Interface（容器网络接口）](https://jimmysong.io/kubernetes-handbook/concepts/cni.html)
[CNI (Container Network Interface)](https://feisky.gitbooks.io/sdn/container/cni/)
[CNI：容器网络接口](https://cizixs.com/2017/05/23/container-network-cni/)

---
title: "client-go 源码分析四：ClientSet"
date: 2020-07-05T17:32:10+08:00
draft: true
categories: ["技术"]
tags: ["k8s"]
---
长恨此身非我有，何时忘却营营。夜阑风静縠纹平。小舟从此逝，江海寄余生。 ——《临江仙·夜归临皋》
<!--more-->
# 简述
通过前面章节知道，`RESTClient`是一种最基础的客户端，使用时需要指定Resource，Group，Version等信息，相对复杂。ClientSet则基于RESTClient对Resource，Group，Version进行了再次封装，增加了对Resource，Group和Version的管理方法，使得开发使用更加简单。

# 使用示例
列出default命名空间下所有的Pod资源对象
{{< highlight go "linenos=inline" >}}
package main

import (
	"context"
	"fmt"

	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"

	apiv1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "/home/zvier/.kube/config")
	if err != nil {
		fmt.Println(err)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}
    // 请求core核心资源组v1资源版本下的Pod资源对象
    // 其内部设置了APIPath，GroupVersion，NegotiatedSerializer数据编解码器
	podClient := clientset.CoreV1().Pods(apiv1.NamespaceDefault)
	list, err := podClient.List(context.TODO(), metav1.ListOptions{Limit: 500})
	if err != nil {
		panic(err)
	}
	for _, d := range list.Items {
		fmt.Printf("NAMESPACE: %+v\t NAME: %v\t STATUS: %v\n", d.Namespace, d.Name, d.Status.Phase)
	}
}
{{< /highlight >}}

# NewForConfig
NewForConfig根据kubeconfig配置，返回一个Clientset指针
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/kubernetes/clientset.go
// NewForConfig creates a new Clientset for the given config.
// If config's RateLimiter is not set and QPS and Burst are acceptable,
// NewForConfig will generate a rate-limiter in configShallowCopy.
func NewForConfig(c *rest.Config) (*Clientset, error) {
    configShallowCopy := *c
    if configShallowCopy.RateLimiter == nil && configShallowCopy.QPS > 0 {
        if configShallowCopy.Burst <= 0 {
            return nil, fmt.Errorf("burst is required to be greater than 0 when RateLimiter is not set and QPS is set to greater than 0")
        }
        configShallowCopy.RateLimiter = flowcontrol.NewTokenBucketRateLimiter(configShallowCopy.QPS, configShallowCopy.Burst)
    }
    var cs Clientset
    var err error
    cs.admissionregistrationV1, err = admissionregistrationv1.NewForConfig(&configShallowCopy)
    if err != nil {
        return nil, err
    }
    cs.admissionregistrationV1beta1, err = admissionregistrationv1beta1.NewForConfig(&configShallowCopy)
    if err != nil {
        return nil, err
    }
    cs.appsV1, err = appsv1.NewForConfig(&configShallowCopy)
    if err != nil {
        return nil, err
    }
    cs.appsV1beta1, err = appsv1beta1.NewForConfig(&configShallowCopy)
    if err != nil {
        return nil, err
    }
    cs.appsV1beta2, err = appsv1beta2.NewForConfig(&configShallowCopy)
    if err != nil {
        return nil, err
    }
    ...
    // corev1 "k8s.io/client-go/kubernetes/typed/core/v1"
    cs.coreV1, err = corev1.NewForConfig(&configShallowCopy)
    if err != nil {
        return nil, err
    }
    ...
    cs.DiscoveryClient = discovery.NewDiscoveryClientForConfigOrDie(c)
    return &cs
}
{{< /highlight >}}

# Clientset
在kubernetes中，每一种Resource都有一个客户端，ClientSet是多个客户端的集合。
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/kubernetes/clientset.go
// Clientset contains the clients for groups. Each group has exactly one
// version included in a Clientset.
type Clientset struct {
    *discovery.DiscoveryClient
    admissionregistrationV1      *admissionregistrationv1.AdmissionregistrationV1Client
    admissionregistrationV1beta1 *admissionregistrationv1beta1.AdmissionregistrationV1beta1Client
    appsV1                       *appsv1.AppsV1Client
    appsV1beta1                  *appsv1beta1.AppsV1beta1Client
    appsV1beta2                  *appsv1beta2.AppsV1beta2Client
    auditregistrationV1alpha1    *auditregistrationv1alpha1.AuditregistrationV1alpha1Client
    authenticationV1             *authenticationv1.AuthenticationV1Client
    authenticationV1beta1        *authenticationv1beta1.AuthenticationV1beta1Client
    authorizationV1              *authorizationv1.AuthorizationV1Client
    authorizationV1beta1         *authorizationv1beta1.AuthorizationV1beta1Client
    autoscalingV1                *autoscalingv1.AutoscalingV1Client
    autoscalingV2beta1           *autoscalingv2beta1.AutoscalingV2beta1Client
    autoscalingV2beta2           *autoscalingv2beta2.AutoscalingV2beta2Client
    batchV1                      *batchv1.BatchV1Client
    batchV1beta1                 *batchv1beta1.BatchV1beta1Client
    batchV2alpha1                *batchv2alpha1.BatchV2alpha1Client
    certificatesV1beta1          *certificatesv1beta1.CertificatesV1beta1Client
    coordinationV1beta1          *coordinationv1beta1.CoordinationV1beta1Client
    coordinationV1               *coordinationv1.CoordinationV1Client
    // corev1 "k8s.io/client-go/kubernetes/typed/core/v1"
    coreV1                       *corev1.CoreV1Client
    discoveryV1alpha1            *discoveryv1alpha1.DiscoveryV1alpha1Client
    discoveryV1beta1             *discoveryv1beta1.DiscoveryV1beta1Client
    eventsV1beta1                *eventsv1beta1.EventsV1beta1Client
    extensionsV1beta1            *extensionsv1beta1.ExtensionsV1beta1Client
    flowcontrolV1alpha1          *flowcontrolv1alpha1.FlowcontrolV1alpha1Client
    networkingV1                 *networkingv1.NetworkingV1Client
    networkingV1beta1            *networkingv1beta1.NetworkingV1beta1Client
    nodeV1alpha1                 *nodev1alpha1.NodeV1alpha1Client
    nodeV1beta1                  *nodev1beta1.NodeV1beta1Client
    policyV1beta1                *policyv1beta1.PolicyV1beta1Client
    rbacV1                       *rbacv1.RbacV1Client
    rbacV1beta1                  *rbacv1beta1.RbacV1beta1Client
    rbacV1alpha1                 *rbacv1alpha1.RbacV1alpha1Client
    schedulingV1alpha1           *schedulingv1alpha1.SchedulingV1alpha1Client
    schedulingV1beta1            *schedulingv1beta1.SchedulingV1beta1Client
    schedulingV1                 *schedulingv1.SchedulingV1Client
    settingsV1alpha1             *settingsv1alpha1.SettingsV1alpha1Client
    storageV1beta1               *storagev1beta1.StorageV1beta1Client
    storageV1                    *storagev1.StorageV1Client
    storageV1alpha1              *storagev1alpha1.StorageV1alpha1Client
    storageV1                    *storagev1.StorageV1Client
    storageV1alpha1              *storagev1alpha1.StorageV1alpha1Client
}
{{< /highlight >}}

# CoreV1Client
CoreV1Client用于与core组v1版本的资源进行交互
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/kubernetes/typed/core/v1/core_client.go
// CoreV1Client is used to interact with features provided by the  group.
type CoreV1Client struct {
    restClient rest.Interface
}
{{< /highlight >}}

# rest.Interface
rest.Interface定义了一组RESTful api方法
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/rest/client.go
// Interface captures the set of operations for generically interacting with Kubernetes REST apis.
type Interface interface {
    GetRateLimiter() flowcontrol.RateLimiter
    Verb(verb string) *Request
    Post() *Request
    Put() *Request
    Patch(pt types.PatchType) *Request
    Get() *Request
    Delete() *Request
    APIVersion() schema.GroupVersion
}
{{< /highlight >}}

# CoreV1Client.Pods
CoreV1Client.Pods返回一个PodInterface实例
{{< highlight go "linenos=inline" >}}
func (c *CoreV1Client) Pods(namespace string) PodInterface {
    return newPods(c, namespace)
}
{{< /highlight >}}

# newPods
newPods返回一个pods对象的指针，该对象实现了PodInterface接口
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/kubernetes/typed/core/v1/pod.go
// newPods returns a Pods
func newPods(c *CoreV1Client, namespace string) *pods {
    return &pods{
        client: c.RESTClient(),
        ns:     namespace,
    }
}
{{< /highlight >}}

# CoreV1Client.RESTClient
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/kubernetes/typed/core/v1/core_client.go
// RESTClient returns a RESTClient that is used to communicate
// with API server by this client implementation.
func (c *CoreV1Client) RESTClient() rest.Interface {
    if c == nil {
        return nil
    }
    return c.restClient
}
{{< /highlight >}}

# pods
pods封装了RESTfulClient和ns
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/kubernetes/typed/core/v1/pod.go
// pods implements PodInterface
type pods struct {
    client rest.Interface
    ns     string
}
{{< /highlight >}}

# pods的增删改查操作
pods的各种增删改查操作其实就是调用了RESTfulClient和apiserver进行交互
## pods.Get
{{< highlight go "linenos=inline" >}}
// Get takes name of the pod, and returns the corresponding pod object, and an error if there is any.
func (c *pods) Get(ctx context.Context, name string, options metav1.GetOptions) (result *v1.Pod, err error) {
    result = &v1.Pod{}
    err = c.client.Get().
        Namespace(c.ns).
        Resource("pods").
        Name(name).
        VersionedParams(&options, scheme.ParameterCodec).
        Do(ctx).
        Into(result)
    return
}
{{< /highlight >}}

## pods.List
{{< highlight go "linenos=inline" >}}
// List takes label and field selectors, and returns the list of Pods that match those selectors.
func (c *pods) List(ctx context.Context, opts metav1.ListOptions) (result *v1.PodList, err error) {
    var timeout time.Duration
    if opts.TimeoutSeconds != nil {
        timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
    }
    result = &v1.PodList{}
    err = c.client.Get().
        Namespace(c.ns).
        Resource("pods").
        VersionedParams(&opts, scheme.ParameterCodec).
        Timeout(timeout).
        Do(ctx).
        Into(result)
    return
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
 14 // Watch returns a watch.Interface that watches the requested pods.
 13 func (c *pods) Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error) {
 12     var timeout time.Duration
 11     if opts.TimeoutSeconds != nil {
 10         timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
  9     }
  8     opts.Watch = true
  7     return c.client.Get().
  6         Namespace(c.ns).
  5         Resource("pods").
  4         VersionedParams(&opts, scheme.ParameterCodec).
  3         Timeout(timeout).
  2         Watch(ctx)
  1 }
{{< /highlight >}}

# NewForConfig
{{< highlight go "linenos=inline" >}}
// NewForConfig creates a new CoreV1Client for the given config.
func NewForConfig(c *rest.Config) (*CoreV1Client, error) {
    config := *c
    if err := setConfigDefaults(&config); err != nil {
        return nil, err
    }
        // rest "k8s.io/client-go/rest"
    client, err := rest.RESTClientFor(&config)
    if err != nil {
        return nil, err
    }
    return &CoreV1Client{client}, nil
}
{{< /highlight >}}

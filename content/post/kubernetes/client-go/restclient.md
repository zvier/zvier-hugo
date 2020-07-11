---
title: "client-go 源码分析三：RESTClient"
date: 2020-07-05T10:34:22+08:00
draft: true
categories: ["技术"]
tags: ["k8s"]
---
<!--more-->
# 简述
RESTClient是k8s.io/client-go中最基础的客户端，通过对HTTP Request进行封装，实现了一套RESTful风格的API。

# 使用示例
实现请求http://127.0.0.1:8080/api/v1/namespaces/default/pods?limit=500
{{< highlight go "linenos=inline" >}}
package main

import (
	"context"
	"fmt"

	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/kubectl/pkg/scheme"

	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func main() {
    // 加载配置
	config, err := clientcmd.BuildConfigFromFlags("", "/home/zvier/.kube/config")
	if err != nil {
        panic(err)
	}

    // 设置HTTP请求路径
	config.APIPath = "api"
    // 设置资源组和资源版本
	config.GroupVersion = &corev1.SchemeGroupVersion
    // 设置数据的编解码器
	config.NegotiatedSerializer = scheme.Codecs

    // 利用config，初始化一个RESTClientFor
	restClient, err := rest.RESTClientFor(config)
	if err != nil {
		panic(err)
	}
	result := &corev1.PodList{}
    // limit
	err = restClient.Get().Namespace("default").Resource("pods").VersionedParams(&metav1.ListOptions{Limit: 500}, scheme.ParameterCodec).Do(context.TODO()).Into(result)
	if err != nil {
		panic(err)
	}
	for _, d := range result.Items {
		fmt.Printf("NAMESPACE: %v\t NAME: %v\t STATUS: %v\n", d.Namespace, d.Name, d.Status.Phase)
	}
}
{{< /highlight >}}

# RESTClientFor
RESTClientFor根据config返回一个RESTClient指针
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/rest/config.go
// RESTClientFor returns a RESTClient that satisfies the requested attributes on a client Config
// object. Note that a RESTClient may require fields that are optional when initializing a Client.
// A RESTClient created by this method is generic - it expects to operate on an API that follows
// the Kubernetes conventions, but may not be the Kubernetes API.
func RESTClientFor(config *Config) (*RESTClient, error) {
    if config.GroupVersion == nil {
        return nil, fmt.Errorf("GroupVersion is required when initializing a RESTClient")
    }
    if config.NegotiatedSerializer == nil {
        return nil, fmt.Errorf("NegotiatedSerializer is required when initializing a RESTClient")
    }

    baseURL, versionedAPIPath, err := defaultServerUrlFor(config)
    if err != nil {
        return nil, err
    }

    transport, err := TransportFor(config)
    if err != nil {
        return nil, err
    }

    var httpClient *http.Client
    if transport != http.DefaultTransport {
        httpClient = &http.Client{Transport: transport}
        if config.Timeout > 0 {
            httpClient.Timeout = config.Timeout
        }
    }

    rateLimiter := config.RateLimiter
    if rateLimiter == nil {
        qps := config.QPS
        if config.QPS == 0.0 {
            qps = DefaultQPS
        }
        burst := config.Burst
        if config.Burst == 0 {
            burst = DefaultBurst
        }
        if qps > 0 {
            rateLimiter = flowcontrol.NewTokenBucketRateLimiter(qps, burst)
        }
    }
    var gv schema.GroupVersion
    if config.GroupVersion != nil {
        gv = *config.GroupVersion
    }
    clientContent := ClientContentConfig{
        AcceptContentTypes: config.AcceptContentTypes,
        ContentType:        config.ContentType,
        GroupVersion:       gv,
        Negotiator:         runtime.NewClientNegotiator(config.NegotiatedSerializer, gv),
    }

    return NewRESTClient(baseURL, versionedAPIPath, clientContent, rateLimiter, httpClient)
}
{{< /highlight >}}

# NewRESTClient
{{< highlight go "linenos=inline" >}}
// NewRESTClient creates a new RESTClient. This client performs generic REST functions
// such as Get, Put, Post, and Delete on specified paths.
func NewRESTClient(baseURL *url.URL, versionedAPIPath string, config ClientContentConfig, rateLimiter flowcontrol.RateLimiter, client *http.Client) (*RESTClient, error) {
    if len(config.ContentType) == 0 {
        config.ContentType = "application/json"
    }

    base := *baseURL
    if !strings.HasSuffix(base.Path, "/") {
        base.Path += "/"
    }
    base.RawQuery = ""
    base.Fragment = ""

    return &RESTClient{
        base:             &base,
        versionedAPIPath: versionedAPIPath,
        content:          config,
        createBackoffMgr: readExpBackoffConfig,
        rateLimiter:      rateLimiter,

        Client: client,
    }, nil
}
{{< /highlight >}}

# RESTClient
{{< highlight go "linenos=inline" >}}
// RESTClient imposes common Kubernetes API conventions on a set of resource paths.
// The baseURL is expected to point to an HTTP or HTTPS path that is the parent
// of one or more resources.  The server should return a decodable API resource
// object, or an api.Status object which contains information about the reason for
// any failure.
//
// Most consumers should use client.New() to get a Kubernetes API client.
type RESTClient struct {
    // base is the root URL for all invocations of the client
    base *url.URL
    // versionedAPIPath is a path segment connecting the base URL to the resource root
    versionedAPIPath string

    // content describes how a RESTClient encodes and decodes responses.
    content ClientContentConfig

    // creates BackoffManager that is passed to requests.
    createBackoffMgr func() BackoffManager

    // rateLimiter is shared among all requests created by this client unless specifically
    // overridden.
    rateLimiter flowcontrol.RateLimiter

    // Set specific behavior of the client.  If not set http.DefaultClient will be used.
    Client *http.Client
}
{{< /highlight >}}

# RateLimiter
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/util/flowcontrol/throttle.go
 type RateLimiter interface {
     // TryAccept returns true if a token is taken immediately. Otherwise,
     // it returns false.
     TryAccept() bool
     // Accept returns once a token becomes available.
     Accept()
     // Stop stops the rate limiter, subsequent calls to CanAccept will return false
     Stop()
     // QPS returns QPS of this rate limiter
     QPS() float32
     // Wait returns nil if a token is taken before the Context is done.
     Wait(ctx context.Context) error
 }
{{< /highlight >}}

# ClientContentConfig
{{< highlight go "linenos=inline" >}}
// ClientContentConfig controls how RESTClient communicates with the server.
//
// TODO: ContentConfig will be updated to accept a Negotiator instead of a
//   NegotiatedSerializer and NegotiatedSerializer will be removed.
type ClientContentConfig struct {
    // AcceptContentTypes specifies the types the client will accept and is optional.
    // If not set, ContentType will be used to define the Accept header
    AcceptContentTypes string
    // ContentType specifies the wire format used to communicate with the server.
    // This value will be set as the Accept header on requests made to the server if
    // AcceptContentTypes is not set, and as the default content type on any object
    // sent to the server. If not set, "application/json" is used.
    ContentType string
    // GroupVersion is the API version to talk to. Must be provided when initializing
    // a RESTClient directly. When initializing a Client, will be set with the default
    // code version. This is used as the default group version for VersionedParams.
    GroupVersion schema.GroupVersion
    // Negotiator is used for obtaining encoders and decoders for multiple
    // supported media types.
    Negotiator runtime.ClientNegotiator
}
{{< /highlight >}}


# RESTClient谓词
RESTClient支持POST、PUT、PATCH、GET、DELETE等请求方法
{{< highlight go "linenos=inline" >}}
func (c *RESTClient) Verb(verb string) *Request {
    return NewRequest(c).Verb(verb)
}

// Post begins a POST request. Short for c.Verb("POST").
func (c *RESTClient) Post() *Request {
    return c.Verb("POST")
}

// Put begins a PUT request. Short for c.Verb("PUT").
func (c *RESTClient) Put() *Request {
    return c.Verb("PUT")
}

// Patch begins a PATCH request. Short for c.Verb("Patch").
func (c *RESTClient) Patch(pt types.PatchType) *Request {
    return c.Verb("PATCH").SetHeader("Content-Type", string(pt))
}

// Get begins a GET request. Short for c.Verb("GET").
func (c *RESTClient) Get() *Request {
    return c.Verb("GET")
}

// Delete begins a DELETE request. Short for c.Verb("DELETE").
func (c *RESTClient) Delete() *Request {
    return c.Verb("DELETE")
}
{{< /highlight >}}

# Request
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/rest/request.go
  37 // Request allows for building up a request to a server in a chained fashion.
  36 // Any errors are stored until the end of your call, so you only have to
  35 // check once.
  34 type Request struct {
  33     c *RESTClient
  32
  31     rateLimiter flowcontrol.RateLimiter
  30     backoff     BackoffManager
  29     timeout     time.Duration
  28     maxRetries  int
  27
  26     // generic components accessible via method setters
  25     verb       string
  24     pathPrefix string
  23     subpath    string
  22     params     url.Values
  21     headers    http.Header
  20
  19     // structural elements of the request that are part of the Kubernetes API conventions
  18     namespace    string
  17     namespaceSet bool
  16     resource     string
  15     resourceName string
  14     subresource  string
  13
  12     // output
  11     err  error
  10     body io.Reader
   9 }
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
  29 // Namespace applies the namespace scope to a request (<resource>/[ns/<namespace>/]<name>)
  28 func (r *Request) Namespace(namespace string) *Request {
  27     if r.err != nil {
  26         return r
  25     }
  24     if r.namespaceSet {
  23         r.err = fmt.Errorf("namespace already set to %q, cannot change to %q", r.namespace, namespace)
  22         return r
  21     }
  20     if msgs := IsValidPathSegmentName(namespace); len(msgs) != 0 {
  19         r.err = fmt.Errorf("invalid namespace %q: %v", namespace, msgs)
  18         return r
  17     }
  16     r.namespaceSet = true
  15     r.namespace = namespace
  14     return r
  13 }
{{< /highlight >}}

# Resource
{{< highlight go "linenos=inline" >}}
// Resource sets the resource to access (<resource>/[ns/<namespace>/]<name>)
func (r *Request) Resource(resource string) *Request {
    if r.err != nil {
        return r
    }
    if len(r.resource) != 0 {
        r.err = fmt.Errorf("resource already set to %q, cannot change to %q", r.resource, resource)
        return r
    }
    if msgs := IsValidPathSegmentName(resource); len(msgs) != 0 {
        r.err = fmt.Errorf("invalid resource %q: %v", resource, msgs)
        return r
    }
    r.resource = resource
    return r
}
{{< /highlight >}}

# VersionedParams
{{< highlight go "linenos=inline" >}}
// VersionedParams will take the provided object, serialize it to a map[string][]string using the
// implicit RESTClient API version and the default parameter codec, and then add those as parameters
// to the request. Use this to provide versioned query parameters from client libraries.
// VersionedParams will not write query parameters that have omitempty set and are empty. If a
// parameter has already been set it is appended to (Params and VersionedParams are additive).
func (r *Request) VersionedParams(obj runtime.Object, codec runtime.ParameterCodec) *Request {
    return r.SpecificallyVersionedParams(obj, codec, r.c.content.GroupVersion)
}

func (r *Request) SpecificallyVersionedParams(obj runtime.Object, codec runtime.ParameterCodec, version schema.GroupVersion) *Request {
    if r.err != nil {
        return r
    }
    params, err := codec.EncodeParameters(obj, version)
    if err != nil {
        r.err = err
        return r
    }
    for k, v := range params {
        if r.params == nil {
            r.params = make(url.Values)
        }
        r.params[k] = append(r.params[k], v...)
    }
    return r
}
{{< /highlight >}}

# Request.DO
Request.DO执行请求，返回kube-apiserver的响应结果
{{< highlight go "linenos=inline" >}}
// Do formats and executes the request. Returns a Result object for easy response
// processing.
//
// Error type:
//  * If the server responds with a status: *errors.StatusError or *errors.UnexpectedObjectError
//  * http.Client.Do errors are returned directly.
func (r *Request) Do(ctx context.Context) Result {
    var result Result
    err := r.request(ctx, func(req *http.Request, resp *http.Response) {
        result = r.transformResponse(resp, req)
    })
    if err != nil {
        return Result{err: err}
    }
    return result
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
  45 // request connects to the server and invokes the provided function when a server response is
  44 // received. It handles retry behavior and up front validation of requests. It will invoke
  43 // fn at most once. It will return an error if a problem occurred prior to connecting to the
  42 // server - the provided function is responsible for handling server errors.
  41 func (r *Request) request(ctx context.Context, fn func(*http.Request, *http.Response)) error {
  40     //Metrics for total request latency
  39     start := time.Now()
  38     defer func() {
  37         metrics.RequestLatency.Observe(r.verb, r.finalURLTemplate(), time.Since(start))
  36     }()
  35
  34     if r.err != nil {
  33         klog.V(4).Infof("Error in request: %v", r.err)
  32         return r.err
  31     }
  30
  29     if err := r.requestPreflightCheck(); err != nil {
  28         return err
  27     }
  26
  25     client := r.c.Client
  24     if client == nil {
  23         client = http.DefaultClient
  22     }
  21
  20     // Throttle the first try before setting up the timeout configured on the
  19     // client. We don't want a throttled client to return timeouts to callers
  18     // before it makes a single request.
  17     if err := r.tryThrottle(ctx); err != nil {
  16         return err
  15     }
  14
  13     if r.timeout > 0 {
  12         var cancel context.CancelFunc
  11         ctx, cancel = context.WithTimeout(ctx, r.timeout)
  10         defer cancel()
   9     }
   8
   7     // Right now we make about ten retry attempts if we get a Retry-After response.
   6     retries := 0
 45     for {
  44
  43         url := r.URL().String()
  42         req, err := http.NewRequest(r.verb, url, r.body)
  41         if err != nil {
  40             return err
  39         }
  38         req = req.WithContext(ctx)
  37         req.Header = r.headers
  36
  35         r.backoff.Sleep(r.backoff.CalculateBackoff(r.URL()))
  34         if retries > 0 {
  33             // We are retrying the request that we already send to apiserver
  32             // at least once before.
  31             // This request should also be throttled with the client-internal rate limiter.
  30             if err := r.tryThrottle(ctx); err != nil {
  29                 return err
  28             }
  27         }
  26         resp, err := client.Do(req)
  25         updateURLMetrics(r, resp, err)
  24         if err != nil {
  23             r.backoff.UpdateBackoff(r.URL(), err, 0)
  22         } else {
  21             r.backoff.UpdateBackoff(r.URL(), err, resp.StatusCode)
  20         }
  19         if err != nil {
  18             // "Connection reset by peer" or "apiserver is shutting down" are usually a transient errors.
  17             // Thus in case of "GET" operations, we simply retry it.
  16             // We are not automatically retrying "write" operations, as
  15             // they are not idempotent.
  14             if r.verb != "GET" {
  13                 return err
  12             }
  45             // For connection errors and apiserver shutdown errors retry.
  44             if net.IsConnectionReset(err) || net.IsProbableEOF(err) {
  43                 // For the purpose of retry, we set the artificial "retry-after" response.
  42                 // TODO: Should we clean the original response if it exists?
  41                 resp = &http.Response{
  40                     StatusCode: http.StatusInternalServerError,
  39                     Header:     http.Header{"Retry-After": []string{"1"}},
  38                     Body:       ioutil.NopCloser(bytes.NewReader([]byte{})),
  37                 }
  36             } else {
  35                 return err
  34             }
  33         }
  32
  31         done := func() bool {
  30             // Ensure the response body is fully read and closed
  29             // before we reconnect, so that we reuse the same TCP
  28             // connection.
  27             defer func() {
  26                 const maxBodySlurpSize = 2 << 10
  25                 if resp.ContentLength <= maxBodySlurpSize {
  24                     io.Copy(ioutil.Discard, &io.LimitedReader{R: resp.Body, N: maxBodySlurpSize})
  23                 }
  22                 resp.Body.Close()
  21             }()
  20
  19             retries++
  18             if seconds, wait := checkWait(resp); wait && retries <= r.maxRetries {
  17                 if seeker, ok := r.body.(io.Seeker); ok && r.body != nil {
  16                     _, err := seeker.Seek(0, 0)
  15                     if err != nil {
  14                         klog.V(4).Infof("Could not retry request, can't Seek() back to beginning of body for %T", r.body)
  13                         fn(req, resp)
  12                         return true
  11                     }
  10                 }
   9
   8                 klog.V(4).Infof("Got a Retry-After %ds response for attempt %d to %v", seconds, retries, url)
   7                 r.backoff.Sleep(time.Duration(seconds) * time.Second)
   6                 return false
   5             }
   4             fn(req, resp)
   3             return true
   2         }()
  23         if done {
  22             return nil
  21         }
  20     }
  19 }

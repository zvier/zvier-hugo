---
title: "docker-docker-client-client"
date: 2019-09-22T19:12:05+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
# 简述
docker引擎的客户端实现，更多API相关的资料可以参考[docker sdk](https://docs.docker.com/develop/sdk/)   
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/docker/docker/client/client.go
{{< /highlight >}}

# docker ps示例
以下是实现<code>docker ps</code>的一个demo
{{< highlight go "linenos=inline" >}}
package main

import (
	"context"
	"fmt"

	"github.com/docker/docker/api/types"
	"github.com/docker/docker/client"
)

func main() {
	cli, err := client.NewClientWithOpts(client.FromEnv)
	if err != nil {
		panic(err)
	}

	containers, err := cli.ContainerList(context.Background(), types.ContainerListOptions{})
	if err != nil {
		panic(err)
	}

	for _, container := range containers {
		fmt.Printf("%s %s\n", container.ID[:10], container.Image)
	}
}
{{< /highlight >}}

# 变量声明
{{< highlight go "linenos=inline" >}}
// ErrRedirect is the error returned by checkRedirect when the request is non-GET.
var ErrRedirect = errors.New("unexpected redirect in response")
{{< /highlight >}}

# Client
Client是docker server的API客户端部分  
{{< highlight go "linenos=inline" >}}
// Client is the API client that performs all operations
// against a docker server.
type Client struct {
	// scheme sets the scheme for the client
	scheme string
	// host holds the server address to connect to
	host string
	// proto holds the client protocol i.e. unix.
	proto string
	// addr holds the client address.
	addr string
	// basePath holds the path to prepend to the requests.
	basePath string
	// client used to send and receive http requests.
	client *http.Client
	// version of the server to talk to.
	version string
	// custom http headers configured by users.
	customHTTPHeaders map[string]string
	// manualOverride is set to true when the version was set by users.
	manualOverride bool

	// negotiateVersion indicates if the client should automatically negotiate
	// the API version to use when making requests. API version negotiation is
	// performed on the first request, after which negotiated is set to "true"
	// so that subsequent requests do not re-negotiate.
	negotiateVersion bool

	// negotiated indicates that API version negotiation took place
	negotiated bool
}
{{< /highlight >}}

# CheckRedirect
{{< highlight go "linenos=inline" >}}
// CheckRedirect specifies the policy for dealing with redirect responses:
// If the request is non-GET return `ErrRedirect`. Otherwise use the last response.

// Go 1.8 changes behavior for HTTP redirects (specifically 301, 307, and 308) in the client .
// The Docker client (and by extension docker API client) can be made to send a request
// like POST /containers//start where what would normally be in the name section of the URL is empty.
// This triggers an HTTP 301 from the daemon.
// In go 1.8 this 301 will be converted to a GET request, and ends up getting a 404 from the daemon.
// This behavior change manifests in the client in that before the 301 was not followed and
// the client did not generate an error, but now results in a message like Error response from daemon: page not found.
func CheckRedirect(req *http.Request, via []*http.Request) error {
	if via[0].Method == http.MethodGet {
		return http.ErrUseLastResponse
	}
	return ErrRedirect
}
{{< /highlight >}}

> 301: 永久性转移(Permanently Moved)  
> 302: 发现(Found)，要求客户端执行临时重定向(Moved Temporarily)。由于这样的重定向是临时的，客户端应当继续向原有地址发送后续请求。新的临时性的URI应当在响应的Location域中返回。
> 307: 临时重定向(Temporary Redirect)  
> 308: 永久重定向(Permanent Redirect)  

## http.MethodGet
{{< highlight go "linenos=inline" >}}
// import "net/http"
const (
    MethodGet     = "GET"
    MethodHead    = "HEAD"
    MethodPost    = "POST"
    MethodPut     = "PUT"
    MethodPatch   = "PATCH" // RFC 5789
    MethodDelete  = "DELETE"
    ...
)
{{< /highlight >}}

> Common HTTP methods.

## http.ErrUseLastResponse
{{< highlight go "linenos=inline" >}}
var ErrUseLastResponse = errors.New("net/http: use last response")
{{< /highlight >}}

> ErrUseLastResponse can be returned by Client.CheckRedirect hooks to control how redirects are processed. If returned, the next request is not sent and the most recent response is returned with its body unclosed.

## http.Client重定向问题
http-client 4.2默认会自动跟随(follow)完成重定向，且大多数时候这种自动跟随也是有意义的，比如自动把HTTP连接重定向到HTTPS连接。  
但是有时我们又不需要这种自动重定向机制，比如常见的POST登陆，毕竟大多数网站登陆的用户名和秘密等都是通过POST方式提交给Server的，POST登陆的步骤如下:
1. 客户端发起POST /login请求
2. 服务器返回302 Found, 并携带Cookie和新的的登陆后的页面 /index
3. 客户端重新发起到 /index的请求

这里如果直接使用http.Post就成功登陆到最终页面/index的话，中间重定向相关的数据是拿不到的，包括Cookie，而实际上，我们需要将上面第二步返回的Cookie拿到，并用作后续API请求的验证。所以得想办法禁止http.Client的自动重定向。

### http.Client中的重定向
以下是golang net/http包的http.Client结构
{{< highlight go "linenos=inline" >}}
type Client struct {
    // Transport specifies the mechanism by which individual
    // HTTP requests are made.
    // If nil, DefaultTransport is used
    Transport RoundTripper

    // CheckRedirect specifies the policy for handling redirects.
    // If CheckRedirect is not nil, the client calls it before
    // following an HTTP redirect. The arguments req and via are
    // the upcoming request and the requests made already, oldest
    // first. If CheckRedirect returns an error, the Client's Get
    // method returns both the previous Response (with its Body
    // closed) and CheckRedirect's error (wrapped in a url.Error)
    // instead of issuing the Request req.
    // As a special case, if CheckRedirect returns ErrUseLastResponse,
    // then the most recent response is returned with its body
    // unclosed, along with a nil error.
    //
    // If CheckRedirect is nil, the Client uses its default policy,
    // which is to stop after 10 consecutive requests.
    CheckRedirect func(req *Request, via []*Request) error
    Jar CookieJar
    Timeout time.Duration
{{< /highlight >}}
其中CheckRedirect字段和重定向相关，它是一个函数类型，用于指定处理重定向的策略，如果这个字段不为空，Client将在每次重定向之前(before following an HTTP redirect)先调用它。如果这个函数返回了一个特殊的ErrUseLastResponse错误，那么将使用最近一次的请求响应作为返回。如果这个字段为空，Client将采用默认的重定向策略，将在最多连续重定向10 次后停止；

如果我们想禁止重定向，不使用默认的http.Get、http.Post等标准库定义好的方法，而是自定义一个带不重定向功能的Client：
{{< highlight go "linenos=inline" >}}
client := &http.Client{
    CheckRedirect: func(req *http.Request, via []*http.Request) error {
        return http.ErrUseLastResponse
    },
}
{{< /highlight >}}
然后再调用这个client的client.Get、client.Post、client.Do等方法。

# NewClientWithOpts
{{< highlight go "linenos=inline" >}}
// NewClientWithOpts initializes a new API client with default values. It takes functors
// to modify values when creating it, like `NewClientWithOpts(WithVersion(…))`
// It also initializes the custom http headers to add to each request.

// It won't send any version information if the version number is empty. It is
// highly recommended that you set a version or your client may break if the
// server is upgraded.
func NewClientWithOpts(ops ...Opt) (*Client, error) {
	client, err := defaultHTTPClient(DefaultDockerHost)
	if err != nil {
		return nil, err
	}
	c := &Client{
		host:    DefaultDockerHost,
		version: api.DefaultVersion,
		client:  client,
		proto:   defaultProto,
		addr:    defaultAddr,
	}

	for _, op := range ops {
		if err := op(c); err != nil {
			return nil, err
		}
	}

	if _, ok := c.client.Transport.(http.RoundTripper); !ok {
		return nil, fmt.Errorf("unable to verify TLS configuration, invalid transport %v", c.client.Transport)
	}
	if c.scheme == "" {
		c.scheme = "http"

		tlsConfig := resolveTLSConfig(c.client.Transport)
		if tlsConfig != nil {
			// TODO(stevvooe): This isn't really the right way to write clients in Go.
			// `NewClient` should probably only take an `*http.Client` and work from there.
			// Unfortunately, the model of having a host-ish/url-thingy as the connection
			// string has us confusing protocol and transport layers. We continue doing
			// this to avoid breaking existing clients but this should be addressed.
			c.scheme = "https"
		}
	}

	return c, nil
}
{{< /highlight >}}

# defaultHTTPClient
{{< highlight go "linenos=inline" >}}
func defaultHTTPClient(host string) (*http.Client, error) {
	url, err := ParseHostURL(host)
	if err != nil {
		return nil, err
	}
	transport := new(http.Transport)
	sockets.ConfigureTransport(transport, url.Scheme, url.Host)
	return &http.Client{
		Transport:     transport,
		CheckRedirect: CheckRedirect,
	}, nil
}
{{< /highlight >}}

# Client.Close
{{< highlight go "linenos=inline" >}}
// Close the transport used by the client
func (cli *Client) Close() error {
	if t, ok := cli.client.Transport.(*http.Transport); ok {
		t.CloseIdleConnections()
	}
	return nil
}
{{< /highlight >}}

// getAPIPath returns the versioned request path to call the api.
// It appends the query parameters to the path if they are not empty.
func (cli *Client) getAPIPath(ctx context.Context, p string, query url.Values) string {
	var apiPath string
	if cli.negotiateVersion && !cli.negotiated {
		cli.NegotiateAPIVersion(ctx)
	}
	if cli.version != "" {
		v := strings.TrimPrefix(cli.version, "v")
		apiPath = path.Join(cli.basePath, "/v"+v, p)
	} else {
		apiPath = path.Join(cli.basePath, p)
	}
	return (&url.URL{Path: apiPath, RawQuery: query.Encode()}).String()
}

// ClientVersion returns the API version used by this client.
func (cli *Client) ClientVersion() string {
	return cli.version
}

// NegotiateAPIVersion queries the API and updates the version to match the
// API version. Any errors are silently ignored. If a manual override is in place,
// either through the `DOCKER_API_VERSION` environment variable, or if the client
// was initialized with a fixed version (`opts.WithVersion(xx)`), no negotiation
// will be performed.
func (cli *Client) NegotiateAPIVersion(ctx context.Context) {
	if !cli.manualOverride {
		ping, _ := cli.Ping(ctx)
		cli.negotiateAPIVersionPing(ping)
	}
}

// NegotiateAPIVersionPing updates the client version to match the Ping.APIVersion
// if the ping version is less than the default version.  If a manual override is
// in place, either through the `DOCKER_API_VERSION` environment variable, or if
// the client was initialized with a fixed version (`opts.WithVersion(xx)`), no
// negotiation is performed.
func (cli *Client) NegotiateAPIVersionPing(p types.Ping) {
	if !cli.manualOverride {
		cli.negotiateAPIVersionPing(p)
	}
}

// negotiateAPIVersionPing queries the API and updates the version to match the
// API version. Any errors are silently ignored.
func (cli *Client) negotiateAPIVersionPing(p types.Ping) {
	// try the latest version before versioning headers existed
	if p.APIVersion == "" {
		p.APIVersion = "1.24"
	}

	// if the client is not initialized with a version, start with the latest supported version
	if cli.version == "" {
		cli.version = api.DefaultVersion
	}

	// if server version is lower than the client version, downgrade
	if versions.LessThan(p.APIVersion, cli.version) {
		cli.version = p.APIVersion
	}

	// Store the results, so that automatic API version negotiation (if enabled)
	// won't be performed on the next request.
	if cli.negotiateVersion {
		cli.negotiated = true
	}
}

// DaemonHost returns the host address used by the client
func (cli *Client) DaemonHost() string {
	return cli.host
}

// HTTPClient returns a copy of the HTTP client bound to the server
func (cli *Client) HTTPClient() *http.Client {
	return &*cli.client
}

# ParseHostURL
{{< highlight go "linenos=inline" >}}
// ParseHostURL parses a url string, validates the string is a host url, and
// returns the parsed URL
func ParseHostURL(host string) (*url.URL, error) {
	protoAddrParts := strings.SplitN(host, "://", 2)
	if len(protoAddrParts) == 1 {
		return nil, fmt.Errorf("unable to parse docker host `%s`", host)
	}

	var basePath string
	proto, addr := protoAddrParts[0], protoAddrParts[1]
	if proto == "tcp" {
		parsed, err := url.Parse("tcp://" + addr)
		if err != nil {
			return nil, err
		}
		addr = parsed.Host
		basePath = parsed.Path
	}
	return &url.URL{
		Scheme: proto,
		Host:   addr,
		Path:   basePath,
	}, nil
}
{{< /highlight >}}
## Parse
Parse parses rawurl into a URL structure.

The rawurl may be relative (a path, without a host) or absolute (starting with a scheme). Trying to parse a hostname and path without a scheme is invalid but may not necessarily return an error, due to parsing ambiguities.
{{< highlight go "linenos=inline" >}}
// import "net/url"
func Parse(rawurl string) (*URL, error)
{{< /highlight >}}

# CustomHTTPHeaders
提取client中存储的自定义http headers
{{< highlight go "linenos=inline" >}}
// CustomHTTPHeaders returns the custom http headers stored by the client.
func (cli *Client) CustomHTTPHeaders() map[string]string {
	m := make(map[string]string)
	for k, v := range cli.customHTTPHeaders {
		m[k] = v
	}
	return m
}
{{< /highlight >}}

# SetCustomHTTPHeaders
设置client的自定义http headers
{{< highlight go "linenos=inline" >}}
// SetCustomHTTPHeaders that will be set on every HTTP request made by the client.
// Deprecated: use WithHTTPHeaders when creating the client.
func (cli *Client) SetCustomHTTPHeaders(headers map[string]string) {
	cli.customHTTPHeaders = headers
}
{{< /highlight >}}

# Dialer
{{< highlight go "linenos=inline" >}}
// Dialer returns a dialer for a raw stream connection, with HTTP/1.1 header, that can be used for proxying the daemon connection.
// Used by `docker dial-stdio` (docker/cli#889).
func (cli *Client) Dialer() func(context.Context) (net.Conn, error) {
	return func(ctx context.Context) (net.Conn, error) {
		if transport, ok := cli.client.Transport.(*http.Transport); ok {
			if transport.DialContext != nil && transport.TLSClientConfig == nil {
				return transport.DialContext(ctx, cli.proto, cli.addr)
			}
		}
		return fallbackDial(cli.proto, cli.addr, resolveTLSConfig(cli.client.Transport))
	}
}
{{< /highlight >}}

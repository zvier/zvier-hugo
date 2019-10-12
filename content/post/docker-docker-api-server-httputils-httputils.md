---
title: "docker-docker-api-server-httputils-httputils"
date: 2019-10-03T15:28:07+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
# 简述
> 不到园林，怎知春色如许——汤显祖«牡丹亭»
<!--more-->

# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/docker/docker/api/server/httputils/httputils.go
{{< /highlight >}}

# APIVersionKey
{{< highlight go "linenos=inline" >}}
// APIVersionKey is the client's requested API version.
type APIVersionKey struct{}
{{< /highlight >}}

# APIFunc
{{< highlight go "linenos=inline" >}}
// APIFunc is an adapter to allow the use of ordinary functions as Docker API endpoints.
// Any function that has the appropriate signature can be registered as an API endpoint (e.g. getVersion).
type APIFunc func(ctx context.Context, w http.ResponseWriter, r *http.Request, vars map[string]string) error
{{< /highlight >}}

# HijackConnection
HijackConnection用于接管HTTP的socket连接，也就是说，接管之后，golang的内置HTTP库和HTTPServer库将不会管理这个socket
连接的生命周期，这个生命周期已经交给Hijacker了。

{{< highlight go "linenos=inline" >}}
// HijackConnection interrupts the http response writer to get the
// underlying connection and operate with it.
func HijackConnection(w http.ResponseWriter) (io.ReadCloser, io.Writer, error) {
	conn, _, err := w.(http.Hijacker).Hijack()
	if err != nil {
		return nil, nil, err
	}
	// Flush the options to make sure the client sets the raw mode
	conn.Write([]byte{})
	return conn, conn, nil
}
{{< /highlight >}}

## http.Hijacker
go语言的Hijacker可以用于接管请求，对请求消息进行转发。
{{< highlight go "linenos=inline" >}}
import net/http
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}

type Hijacker interface {
    // Hijack lets the caller take over the connection.
    // After a call to Hijack the HTTP server library
    // will not do anything else with the connection.
    //
    // It becomes the caller's responsibility to manage
    // and close the connection.
    //
    // The returned net.Conn may have read or write deadlines
    // already set, depending on the configuration of the
    // Server. It is the caller's responsibility to set
    // or clear those deadlines as needed.
    //
    // The returned bufio.Reader may contain unprocessed buffered
    // data from the client.
    //
    // After a call to Hijack, the original Request.Body must not
    // be used. The original Request's Context remains valid and
    // is not canceled until the Request's ServeHTTP method
    // returns.
    Hijack() (net.Conn, *bufio.ReadWriter, error)
}
{{< /highlight >}}
The Hijacker interface is implemented by ResponseWriters that allow an HTTP handler to take over the connection.

The default ResponseWriter for HTTP/1.x connections supports Hijacker, but HTTP/2 connections intentionally do not. ResponseWriter wrappers may also not support Hijacker. Handlers should always test for this ability at runtime.

## Hijacker的作用
HTTP本身属于Request-Response
模型，一般不需要接管connection，但有一下两个场景需要考虑接管连接
1. Grpc Stream
GRPC 中有四种类型的 RPC，同时 GRPC 是构建在 HTTP/2.0 之上的，那么有没有办法可以通过 HTTP/1.1 来支持 GRPC 的 stream rpc 呢？这里其实就可以通过 hijack 的黑科技来实现，将 Client 和 Server 两端进行 hijack 一番，其实就有点类似于在 TCP 之上通信了。

2. Websocket 管理
Websocket 其实也是有点类似，因为 Websocket 第一阶段走的是普通的 HTTP，后面马上就升级为 Websocket 协议了，所以如果你希望作为中间人操作一些事情的话，那么 hijack 或许是一个很重要的选择。

# CloseStreams
{{< highlight go "linenos=inline" >}}
// CloseStreams ensures that a list for http streams are properly closed.
func CloseStreams(streams ...interface{}) {
	for _, stream := range streams {
		if tcpc, ok := stream.(interface {
			CloseWrite() error
		}); ok {
			tcpc.CloseWrite()
		} else if closer, ok := stream.(io.Closer); ok {
			closer.Close()
		}
	}
}
{{< /highlight >}}

// CheckForJSON makes sure that the request's Content-Type is application/json.
func CheckForJSON(r *http.Request) error {
	ct := r.Header.Get("Content-Type")

	// No Content-Type header is ok as long as there's no Body
	if ct == "" {
		if r.Body == nil || r.ContentLength == 0 {
			return nil
		}
	}

	// Otherwise it better be json
	if matchesContentType(ct, "application/json") {
		return nil
	}
	return errdefs.InvalidParameter(errors.Errorf("Content-Type specified (%s) must be 'application/json'", ct))
}

// ParseForm ensures the request form is parsed even with invalid content types.
// If we don't do this, POST method without Content-type (even with empty body) will fail.
func ParseForm(r *http.Request) error {
	if r == nil {
		return nil
	}
	if err := r.ParseForm(); err != nil && !strings.HasPrefix(err.Error(), "mime:") {
		return errdefs.InvalidParameter(err)
	}
	return nil
}

// VersionFromContext returns an API version from the context using APIVersionKey.
// It panics if the context value does not have version.Version type.
func VersionFromContext(ctx context.Context) string {
	if ctx == nil {
		return ""
	}

	if val := ctx.Value(APIVersionKey{}); val != nil {
		return val.(string)
	}

	return ""
}

// MakeErrorHandler makes an HTTP handler that decodes a Docker error and
// returns it in the response.
func MakeErrorHandler(err error) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		statusCode := errdefs.GetHTTPErrorStatusCode(err)
		vars := mux.Vars(r)
		if apiVersionSupportsJSONErrors(vars["version"]) {
			response := &types.ErrorResponse{
				Message: err.Error(),
			}
			WriteJSON(w, statusCode, response)
		} else {
			http.Error(w, status.Convert(err).Message(), statusCode)
		}
	}
}

func apiVersionSupportsJSONErrors(version string) bool {
	const firstAPIVersionWithJSONErrors = "1.23"
	return version == "" || versions.GreaterThan(version, firstAPIVersionWithJSONErrors)
}

// matchesContentType validates the content type against the expected one
func matchesContentType(contentType, expectedType string) bool {
	mimetype, _, err := mime.ParseMediaType(contentType)
	if err != nil {
		logrus.Errorf("Error parsing media type: %s error: %v", contentType, err)
	}
	return err == nil && mimetype == expectedType
}

---
title: "docker-docker-client-request"
date: 2019-09-22T11:12:01+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
# 简述
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/docker/docker/client/request.go
{{< /highlight >}}

# serverResponse
{{< highlight go "linenos=inline" >}}
// serverResponse is a wrapper for http API responses.
type serverResponse struct {
	body       io.ReadCloser
	header     http.Header
	statusCode int
	reqURL     *url.URL
}
{{< /highlight >}}

# Client.head
{{< highlight go "linenos=inline" >}}
// head sends an http request to the docker API using the method HEAD.
func (cli *Client) head(ctx context.Context, path string, query url.Values, headers map[string][]string) (serverResponse, error) {
	return cli.sendRequest(ctx, "HEAD", path, query, nil, headers)
}
{{< /highlight >}}

# Client.get
{{< highlight go "linenos=inline" >}}
// get sends an http request to the docker API using the method GET with a specific Go context.
func (cli *Client) get(ctx context.Context, path string, query url.Values, headers map[string][]string) (serverResponse, error) {
	return cli.sendRequest(ctx, "GET", path, query, nil, headers)
}
{{< /highlight >}}

# Client.post
{{< highlight go "linenos=inline" >}}
// post sends an http request to the docker API using the method POST with a specific Go context.
func (cli *Client) post(ctx context.Context, path string, query url.Values, obj interface{}, headers map[string][]string) (serverResponse, error) {
	body, headers, err := encodeBody(obj, headers)
	if err != nil {
		return serverResponse{}, err
	}
	return cli.sendRequest(ctx, "POST", path, query, body, headers)
}
{{< /highlight >}}

# Client.postRaw
{{< highlight go "linenos=inline" >}}
func (cli *Client) postRaw(ctx context.Context, path string, query url.Values, body io.Reader, headers map[string][]string) (serverResponse, error) {
	return cli.sendRequest(ctx, "POST", path, query, body, headers)
}
{{< /highlight >}}

# Client.put
{{< highlight go "linenos=inline" >}}
// put sends an http request to the docker API using the method PUT.
func (cli *Client) put(ctx context.Context, path string, query url.Values, obj interface{}, headers map[string][]string) (serverResponse, error) {
	body, headers, err := encodeBody(obj, headers)
	if err != nil {
		return serverResponse{}, err
	}
	return cli.sendRequest(ctx, "PUT", path, query, body, headers)
}
{{< /highlight >}}

# Client.putRaw
{{< highlight go "linenos=inline" >}}
// putRaw sends an http request to the docker API using the method PUT.
func (cli *Client) putRaw(ctx context.Context, path string, query url.Values, body io.Reader, headers map[string][]string) (serverResponse, error) {
	return cli.sendRequest(ctx, "PUT", path, query, body, headers)
}
{{< /highlight >}}

# Client.delete
{{< highlight go "linenos=inline" >}}
// delete sends an http request to the docker API using the method DELETE.
func (cli *Client) delete(ctx context.Context, path string, query url.Values, headers map[string][]string) (serverResponse, error) {
	return cli.sendRequest(ctx, "DELETE", path, query, nil, headers)
}
{{< /highlight >}}

# headers
{{< highlight go "linenos=inline" >}}
type headers map[string][]string
{{< /highlight >}}

# encodeBody
{{< highlight go "linenos=inline" >}}
func encodeBody(obj interface{}, headers headers) (io.Reader, headers, error) {
	if obj == nil {
		return nil, headers, nil
	}

	body, err := encodeData(obj)
	if err != nil {
		return nil, headers, err
	}
	if headers == nil {
		headers = make(map[string][]string)
	}
	headers["Content-Type"] = []string{"application/json"}
	return body, headers, nil
}
{{< /highlight >}}

# Client.buildRequest
{{< highlight go "linenos=inline" >}}
func (cli *Client) buildRequest(method, path string, body io.Reader, headers headers) (*http.Request, error) {
	expectedPayload := (method == "POST" || method == "PUT")
	if expectedPayload && body == nil {
		body = bytes.NewReader([]byte{})
	}

	req, err := http.NewRequest(method, path, body)
	if err != nil {
		return nil, err
	}
	req = cli.addHeaders(req, headers)

	if cli.proto == "unix" || cli.proto == "npipe" {
		// For local communications, it doesn't matter what the host is. We just
		// need a valid and meaningful host name. (See #189)
		req.Host = "docker"
	}

	req.URL.Host = cli.addr
	req.URL.Scheme = cli.scheme

	if expectedPayload && req.Header.Get("Content-Type") == "" {
		req.Header.Set("Content-Type", "text/plain")
	}
	return req, nil
}
{{< /highlight >}}

# Client.sendRequest
{{< highlight go "linenos=inline" >}}
func (cli *Client) sendRequest(ctx context.Context, method, path string, query url.Values, body io.Reader, headers headers) (serverResponse, error) {
	req, err := cli.buildRequest(method, cli.getAPIPath(ctx, path, query), body, headers)
	if err != nil {
		return serverResponse{}, err
	}
	resp, err := cli.doRequest(ctx, req)
	if err != nil {
		return resp, errdefs.FromStatusCode(err, resp.statusCode)
	}
	err = cli.checkResponseErr(resp)
	return resp, errdefs.FromStatusCode(err, resp.statusCode)
}
{{< /highlight >}}

# Client.doRequest
{{< highlight go "linenos=inline" >}}
func (cli *Client) doRequest(ctx context.Context, req *http.Request) (serverResponse, error) {
	serverResp := serverResponse{statusCode: -1, reqURL: req.URL}

	req = req.WithContext(ctx)
	resp, err := cli.client.Do(req)
	if err != nil {
		if cli.scheme != "https" && strings.Contains(err.Error(), "malformed HTTP response") {
			return serverResp, fmt.Errorf("%v.\n* Are you trying to connect to a TLS-enabled daemon without TLS?", err)
		}

		if cli.scheme == "https" && strings.Contains(err.Error(), "bad certificate") {
			return serverResp, errors.Wrap(err, "The server probably has client authentication (--tlsverify) enabled. Please check your TLS client certification settings")
		}

		// Don't decorate context sentinel errors; users may be comparing to
		// them directly.
		switch err {
		case context.Canceled, context.DeadlineExceeded:
			return serverResp, err
		}

		if nErr, ok := err.(*url.Error); ok {
			if nErr, ok := nErr.Err.(*net.OpError); ok {
				if os.IsPermission(nErr.Err) {
					return serverResp, errors.Wrapf(err, "Got permission denied while trying to connect to the Docker daemon socket at %v", cli.host)
				}
			}
		}

		if err, ok := err.(net.Error); ok {
			if err.Timeout() {
				return serverResp, ErrorConnectionFailed(cli.host)
			}
			if !err.Temporary() {
				if strings.Contains(err.Error(), "connection refused") || strings.Contains(err.Error(), "dial unix") {
					return serverResp, ErrorConnectionFailed(cli.host)
				}
			}
		}

		// Although there's not a strongly typed error for this in go-winio,
		// lots of people are using the default configuration for the docker
		// daemon on Windows where the daemon is listening on a named pipe
		// `//./pipe/docker_engine, and the client must be running elevated.
		// Give users a clue rather than the not-overly useful message
		// such as `error during connect: Get http://%2F%2F.%2Fpipe%2Fdocker_engine/v1.26/info:
		// open //./pipe/docker_engine: The system cannot find the file specified.`.
		// Note we can't string compare "The system cannot find the file specified" as
		// this is localised - for example in French the error would be
		// `open //./pipe/docker_engine: Le fichier spécifié est introuvable.`
		if strings.Contains(err.Error(), `open //./pipe/docker_engine`) {
			err = errors.New(err.Error() + " In the default daemon configuration on Windows, the docker client must be run elevated to connect. This error may also indicate that the docker daemon is not running.")
		}

		return serverResp, errors.Wrap(err, "error during connect")
	}

	if resp != nil {
		serverResp.statusCode = resp.StatusCode
		serverResp.body = resp.Body
		serverResp.header = resp.Header
	}
	return serverResp, nil
}
{{< /highlight >}}

# Client.checkResponseErr
{{< highlight go "linenos=inline" >}}
func (cli *Client) checkResponseErr(serverResp serverResponse) error {
	if serverResp.statusCode >= 200 && serverResp.statusCode < 400 {
		return nil
	}

	var body []byte
	var err error
	if serverResp.body != nil {
		bodyMax := 1 * 1024 * 1024 // 1 MiB
		bodyR := &io.LimitedReader{
			R: serverResp.body,
			N: int64(bodyMax),
		}
		body, err = ioutil.ReadAll(bodyR)
		if err != nil {
			return err
		}
		if bodyR.N == 0 {
			return fmt.Errorf("request returned %s with a message (> %d bytes) for API route and version %s, check if the server supports the requested API version", http.StatusText(serverResp.statusCode), bodyMax, serverResp.reqURL)
		}
	}
	if len(body) == 0 {
		return fmt.Errorf("request returned %s for API route and version %s, check if the server supports the requested API version", http.StatusText(serverResp.statusCode), serverResp.reqURL)
	}

	var ct string
	if serverResp.header != nil {
		ct = serverResp.header.Get("Content-Type")
	}

	var errorMessage string
	if (cli.version == "" || versions.GreaterThan(cli.version, "1.23")) && ct == "application/json" {
		var errorResponse types.ErrorResponse
		if err := json.Unmarshal(body, &errorResponse); err != nil {
			return errors.Wrap(err, "Error reading JSON")
		}
		errorMessage = strings.TrimSpace(errorResponse.Message)
	} else {
		errorMessage = strings.TrimSpace(string(body))
	}

	return errors.Wrap(errors.New(errorMessage), "Error response from daemon")
}
{{< /highlight >}}

# Client.addHeaders
遍历客户端所有自定义的HTTP Headers，并设置到req.Header中  
{{< highlight go "linenos=inline" >}}
func (cli *Client) addHeaders(req *http.Request, headers headers) *http.Request {
	// Add CLI Config's HTTP Headers BEFORE we set the Docker headers
	// then the user can't change OUR headers
	for k, v := range cli.customHTTPHeaders {
		if versions.LessThan(cli.version, "1.25") && k == "User-Agent" {
			continue
		}
		req.Header.Set(k, v)
	}

	if headers != nil {
		for k, v := range headers {
			req.Header[k] = v
		}
	}
	return req
}
{{< /highlight >}}

## http.Header
{{< highlight go "linenos=inline" >}}
// import "net/http"
type Header map[string][]string
{{< /highlight >}}

> A Header represents the key-value pairs in an HTTP header.
The keys should be in canonical form, as returned by CanonicalHeaderKey.

## http.Header.Set
{{< highlight go "linenos=inline" >}}
// import "net/http"
func (h Header) Set(key, value string)
{{< /highlight >}}

> Set sets the header entries associated with key to the single element value. It replaces any existing values associated with key. The key is case insensitive; it is canonicalized by textproto.CanonicalMIMEHeaderKey. To use non-canonical keys, assign to the map directly.

## http.Header.Add
{{< highlight go "linenos=inline" >}}
// import "net/http"
func (h Header) Add(key, value string)
{{< /highlight >}}
Add adds the key, value pair to the header. It appends to any existing values associated with key. The key is case insensitive; it is canonicalized by CanonicalHeaderKey.

## 引用说明
1. 客户端自定义HTTPHeaders: [Client.customHTTPHeaders](http://www.zvier.top/post/docker-docker-client-client/#client)

# encodeData
{{< highlight go "linenos=inline" >}}
func encodeData(data interface{}) (*bytes.Buffer, error) {
	params := bytes.NewBuffer(nil)
	if data != nil {
		if err := json.NewEncoder(params).Encode(data); err != nil {
			return nil, err
		}
	}
	return params, nil
}
{{< /highlight >}}
## bytes.NewBuffer
{{< highlight go "linenos=inline" >}}
// import "bytes"
func NewBuffer(buf []byte) *Buffer
{{< /highlight >}}

> NewBuffer creates and initializes a new Buffer using buf as its initial contents. The new Buffer takes ownership of buf, and the caller should not use buf after this call. NewBuffer is intended to prepare a Buffer to read existing data. It can also be used to set the initial size of the internal buffer for writing. To do that, buf should have the desired capacity but a length of zero.

> In most cases, new(Buffer) (or just declaring a Buffer variable) is sufficient to initialize a Buffer.

## json.NewEncoder
{{< highlight go "linenos=inline" >}}
// import "encoding/json"
func NewEncoder(w io.Writer) *Encoder
{{< /highlight >}}

> NewEncoder returns a new encoder that writes to w.

## Encoder.Encode
{{< highlight go "linenos=inline" >}}
// import "encoding/json"
func (enc *Encoder) Encode(v interface{}) error
{{< /highlight >}}

> Encode writes the JSON encoding of v to the stream, followed by a newline character.

# ensureReaderClosed
ensureReaderClosed会"喝光"当前response.body中的512个字节，并关闭之，以便继续复用连接  
{{< highlight go "linenos=inline" >}}
func ensureReaderClosed(response serverResponse) {
	if response.body != nil {
        // Drain up(流干，喝光, 喝干, 用光, 耗尽) to 512 bytes
        // and close the body to let the Transport reuse the connection
		io.CopyN(ioutil.Discard, response.body, 512)
		response.body.Close()
	}
}
{{< /highlight >}}

## io.CpoyN
{{< highlight go "linenos=inline" >}}
// import "io"
func CopyN(dst Writer, src Reader, n int64) (written int64, err error)
{{< /highlight >}}

> CopyN copies n bytes (or until an error) from src to dst. It returns the number of bytes copied and the earliest error encountered while copying. On return, written == n if and only if err == nil.
If dst implements the ReaderFrom interface, the copy is implemented using it.

## ioutil.Discard
{{< highlight go "linenos=inline" >}}
// import "io/ioutil"
var Discard io.Writer = devNull(0)
{{< /highlight >}}

> Discard is an io.Writer on which all Write calls succeed without doing anything.

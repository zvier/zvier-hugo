---
title: "fasthttp-uri"
date: 2019-10-13T06:29:45+08:00
draft: true
categories: ["技术"]
tags: ["fasthttp"]
---
# 简述
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github/fasthttp/uri.go
{{< /highlight >}}

# URI
URI（Uniform Resource Identifier，统一资源标识符）是一个指向资源的字符串，用于唯一地标识一个资源，URI就像是定义了一个规范，要求能满足对资源的唯一标识，具体怎么表并不是很关心。比如常用的形式有通过URL上来指定Web上资源文件的具体位置，通过URN在给定的命名空间用名字指向具体的资源，如：书本的ISBN。

可以说URL和URN都是URI的子集，或者实现。URI的抽象层次更高，是一个字符串文本标准。

> A URI can be further classified as a locator, a name, or both. 

# URL
URL（Uniform Resource Location，统一资源定位符），URI的其中一种常见形式，一是种具体的URI。从Location来看，可定位是其关键特征，URL的locator不仅可以唯一标识一个资源，还指明来如何定位(locate)这个资源。所以说，如何区分一个URI是不是URL，可以从它是否具备可定位的机制。

> The term "Uniform Resource Locator" (URL) refers to the subset of URIs that, in addition to identifying a resource, provide a means of locating the resource by describing its primary access mechanism (e.g., its network "location"). 

URL的格式如下:
1. 协议
2. 存放资源的主机IP和地址
3. 主机资源的具体位置

# URN
URL（Uniform Resource
Name，统一资源名字），也是URI的一种具体形式，通过一个具体的名字来唯一标识一个资源，但它不具备如何定位该资源的描述，通常以urn开头，如:
{{< highlight go "linenos=inline" >}}
urn:isbn:0451450523 to identify a book by its ISBN number.
urn:uuid:6e8bc430-9c3a-11d9-9669-0800200c9a66 a globally unique identifier
urn:publishing:book - An XML namespace that identifies the document as a type of book.
{{< /highlight >}}

# AcquireURI
从URI池中获取一个URI实例  
{{< highlight go "linenos=inline" >}}
// AcquireURI returns an empty URI instance from the pool.
//
// Release the URI with ReleaseURI after the URI is no longer needed.
// This allows reducing GC load.
func AcquireURI() *URI {
	return uriPool.Get().(*URI)
}
{{< /highlight >}}

# ReleaseURI
ReleaseURI重置URI后归还给资源池  
{{< highlight go "linenos=inline" >}}
// ReleaseURI releases the URI acquired via AcquireURI.
//
// The released URI mustn't be used after releasing it, otherwise data races
// may occur.
func ReleaseURI(u *URI) {
	u.Reset()
	uriPool.Put(u)
}
{{< /highlight >}}

# uriPool
{{< highlight go "linenos=inline" >}}
var uriPool = &sync.Pool{
	New: func() interface{} {
		return &URI{}
	},
}
{{< /highlight >}}

# URI
{{< highlight go "linenos=inline" >}}
// URI represents URI :) .

// It is forbidden copying URI instances. Create new instance and use CopyTo
// instead.

// URI instance MUST NOT be used from concurrently running goroutines.
type URI struct {
	noCopy noCopy

	pathOriginal []byte
	scheme       []byte
	path         []byte
	queryString  []byte
	hash         []byte
	host         []byte

	queryArgs       Args
	parsedQueryArgs bool

	fullURI    []byte
	requestURI []byte

	h *RequestHeader
}
{{< /highlight >}}

# URI.CopyTo
{{< highlight go "linenos=inline" >}}
// CopyTo copies uri contents to dst.
func (u *URI) CopyTo(dst *URI) {
	dst.Reset()
	dst.pathOriginal = append(dst.pathOriginal[:0], u.pathOriginal...)
	dst.scheme = append(dst.scheme[:0], u.scheme...)
	dst.path = append(dst.path[:0], u.path...)
	dst.queryString = append(dst.queryString[:0], u.queryString...)
	dst.hash = append(dst.hash[:0], u.hash...)
	dst.host = append(dst.host[:0], u.host...)

	u.queryArgs.CopyTo(&dst.queryArgs)
	dst.parsedQueryArgs = u.parsedQueryArgs

	// fullURI and requestURI shouldn't be copied, since they are created
	// from scratch on each FullURI() and RequestURI() call.
	dst.h = u.h
}
{{< /highlight >}}

# URI.Hash
URI.Hash用于提取URI中的hash字段  
{{< highlight go "linenos=inline" >}}
// Hash returns URI hash, i.e. qwe of http://aaa.com/foo/bar?baz=123#qwe .
//
// The returned value is valid until the next URI method call.
func (u *URI) Hash() []byte {
	return u.hash
}
{{< /highlight >}}

# URI.SetHash
{{< highlight go "linenos=inline" >}}
// SetHash sets URI hash.
func (u *URI) SetHash(hash string) {
	u.hash = append(u.hash[:0], hash...)
}
{{< /highlight >}}

# URI.SetHashBytes
{{< highlight go "linenos=inline" >}}
// SetHashBytes sets URI hash.
func (u *URI) SetHashBytes(hash []byte) {
	u.hash = append(u.hash[:0], hash...)
}
{{< /highlight >}}

# URI.QueryString
{{< highlight go "linenos=inline" >}}
// QueryString returns URI query string,
// i.e. baz=123 of http://aaa.com/foo/bar?baz=123#qwe .
//
// The returned value is valid until the next URI method call.
func (u *URI) QueryString() []byte {
	return u.queryString
}
{{< /highlight >}}

// SetQueryString sets URI query string.
func (u *URI) SetQueryString(queryString string) {
	u.queryString = append(u.queryString[:0], queryString...)
	u.parsedQueryArgs = false
}

// SetQueryStringBytes sets URI query string.
func (u *URI) SetQueryStringBytes(queryString []byte) {
	u.queryString = append(u.queryString[:0], queryString...)
	u.parsedQueryArgs = false
}

// Path returns URI path, i.e. /foo/bar of http://aaa.com/foo/bar?baz=123#qwe .
//
// The returned path is always urldecoded and normalized,
// i.e. '//f%20obar/baz/../zzz' becomes '/f obar/zzz'.
//
// The returned value is valid until the next URI method call.
func (u *URI) Path() []byte {
	path := u.path
	if len(path) == 0 {
		path = strSlash
	}
	return path
}

// SetPath sets URI path.
func (u *URI) SetPath(path string) {
	u.pathOriginal = append(u.pathOriginal[:0], path...)
	u.path = normalizePath(u.path, u.pathOriginal)
}

// SetPathBytes sets URI path.
func (u *URI) SetPathBytes(path []byte) {
	u.pathOriginal = append(u.pathOriginal[:0], path...)
	u.path = normalizePath(u.path, u.pathOriginal)
}

// PathOriginal returns the original path from requestURI passed to URI.Parse().
//
// The returned value is valid until the next URI method call.
func (u *URI) PathOriginal() []byte {
	return u.pathOriginal
}

// Scheme returns URI scheme, i.e. http of http://aaa.com/foo/bar?baz=123#qwe .
//
// Returned scheme is always lowercased.
//
// The returned value is valid until the next URI method call.
func (u *URI) Scheme() []byte {
	scheme := u.scheme
	if len(scheme) == 0 {
		scheme = strHTTP
	}
	return scheme
}

// SetScheme sets URI scheme, i.e. http, https, ftp, etc.
func (u *URI) SetScheme(scheme string) {
	u.scheme = append(u.scheme[:0], scheme...)
	lowercaseBytes(u.scheme)
}

// SetSchemeBytes sets URI scheme, i.e. http, https, ftp, etc.
func (u *URI) SetSchemeBytes(scheme []byte) {
	u.scheme = append(u.scheme[:0], scheme...)
	lowercaseBytes(u.scheme)
}

// Reset clears uri.
func (u *URI) Reset() {
	u.pathOriginal = u.pathOriginal[:0]
	u.scheme = u.scheme[:0]
	u.path = u.path[:0]
	u.queryString = u.queryString[:0]
	u.hash = u.hash[:0]

	u.host = u.host[:0]
	u.queryArgs.Reset()
	u.parsedQueryArgs = false

	// There is no need in u.fullURI = u.fullURI[:0], since full uri
	// is calculated on each call to FullURI().

	// There is no need in u.requestURI = u.requestURI[:0], since requestURI
	// is calculated on each call to RequestURI().

	u.h = nil
}

// Host returns host part, i.e. aaa.com of http://aaa.com/foo/bar?baz=123#qwe .
//
// Host is always lowercased.
func (u *URI) Host() []byte {
	if len(u.host) == 0 && u.h != nil {
		u.host = append(u.host[:0], u.h.Host()...)
		lowercaseBytes(u.host)
		u.h = nil
	}
	return u.host
}

// SetHost sets host for the uri.
func (u *URI) SetHost(host string) {
	u.host = append(u.host[:0], host...)
	lowercaseBytes(u.host)
}

// SetHostBytes sets host for the uri.
func (u *URI) SetHostBytes(host []byte) {
	u.host = append(u.host[:0], host...)
	lowercaseBytes(u.host)
}

// Parse initializes URI from the given host and uri.
//
// host may be nil. In this case uri must contain fully qualified uri,
// i.e. with scheme and host. http is assumed if scheme is omitted.
//
// uri may contain e.g. RequestURI without scheme and host if host is non-empty.
func (u *URI) Parse(host, uri []byte) {
	u.parse(host, uri, nil)
}

func (u *URI) parseQuick(uri []byte, h *RequestHeader, isTLS bool) {
	u.parse(nil, uri, h)
	if isTLS {
		u.scheme = append(u.scheme[:0], strHTTPS...)
	}
}

func (u *URI) parse(host, uri []byte, h *RequestHeader) {
	u.Reset()
	u.h = h

	scheme, host, uri := splitHostURI(host, uri)
	u.scheme = append(u.scheme, scheme...)
	lowercaseBytes(u.scheme)
	u.host = append(u.host, host...)
	lowercaseBytes(u.host)

	b := uri
	queryIndex := bytes.IndexByte(b, '?')
	fragmentIndex := bytes.IndexByte(b, '#')
	// Ignore query in fragment part
	if fragmentIndex >= 0 && queryIndex > fragmentIndex {
		queryIndex = -1
	}

	if queryIndex < 0 && fragmentIndex < 0 {
		u.pathOriginal = append(u.pathOriginal, b...)
		u.path = normalizePath(u.path, u.pathOriginal)
		return
	}

	if queryIndex >= 0 {
		// Path is everything up to the start of the query
		u.pathOriginal = append(u.pathOriginal, b[:queryIndex]...)
		u.path = normalizePath(u.path, u.pathOriginal)

		if fragmentIndex < 0 {
			u.queryString = append(u.queryString, b[queryIndex+1:]...)
		} else {
			u.queryString = append(u.queryString, b[queryIndex+1:fragmentIndex]...)
			u.hash = append(u.hash, b[fragmentIndex+1:]...)
		}
		return
	}

	// fragmentIndex >= 0 && queryIndex < 0
	// Path is up to the start of fragment
	u.pathOriginal = append(u.pathOriginal, b[:fragmentIndex]...)
	u.path = normalizePath(u.path, u.pathOriginal)
	u.hash = append(u.hash, b[fragmentIndex+1:]...)
}

func normalizePath(dst, src []byte) []byte {
	dst = dst[:0]
	dst = addLeadingSlash(dst, src)
	dst = decodeArgAppendNoPlus(dst, src)

	// remove duplicate slashes
	b := dst
	bSize := len(b)
	for {
		n := bytes.Index(b, strSlashSlash)
		if n < 0 {
			break
		}
		b = b[n:]
		copy(b, b[1:])
		b = b[:len(b)-1]
		bSize--
	}
	dst = dst[:bSize]

	// remove /./ parts
	b = dst
	for {
		n := bytes.Index(b, strSlashDotSlash)
		if n < 0 {
			break
		}
		nn := n + len(strSlashDotSlash) - 1
		copy(b[n:], b[nn:])
		b = b[:len(b)-nn+n]
	}

	// remove /foo/../ parts
	for {
		n := bytes.Index(b, strSlashDotDotSlash)
		if n < 0 {
			break
		}
		nn := bytes.LastIndexByte(b[:n], '/')
		if nn < 0 {
			nn = 0
		}
		n += len(strSlashDotDotSlash) - 1
		copy(b[nn:], b[n:])
		b = b[:len(b)-n+nn]
	}

	// remove trailing /foo/..
	n := bytes.LastIndex(b, strSlashDotDot)
	if n >= 0 && n+len(strSlashDotDot) == len(b) {
		nn := bytes.LastIndexByte(b[:n], '/')
		if nn < 0 {
			return strSlash
		}
		b = b[:nn+1]
	}

	return b
}

// RequestURI returns RequestURI - i.e. URI without Scheme and Host.
func (u *URI) RequestURI() []byte {
	dst := appendQuotedPath(u.requestURI[:0], u.Path())
	if u.queryArgs.Len() > 0 {
		dst = append(dst, '?')
		dst = u.queryArgs.AppendBytes(dst)
	} else if len(u.queryString) > 0 {
		dst = append(dst, '?')
		dst = append(dst, u.queryString...)
	}
	if len(u.hash) > 0 {
		dst = append(dst, '#')
		dst = append(dst, u.hash...)
	}
	u.requestURI = dst
	return u.requestURI
}

// LastPathSegment returns the last part of uri path after '/'.
//
// Examples:
//
//    * For /foo/bar/baz.html path returns baz.html.
//    * For /foo/bar/ returns empty byte slice.
//    * For /foobar.js returns foobar.js.
func (u *URI) LastPathSegment() []byte {
	path := u.Path()
	n := bytes.LastIndexByte(path, '/')
	if n < 0 {
		return path
	}
	return path[n+1:]
}

// Update updates uri.
//
// The following newURI types are accepted:
//
//     * Absolute, i.e. http://foobar.com/aaa/bb?cc . In this case the original
//       uri is replaced by newURI.
//     * Absolute without scheme, i.e. //foobar.com/aaa/bb?cc. In this case
//       the original scheme is preserved.
//     * Missing host, i.e. /aaa/bb?cc . In this case only RequestURI part
//       of the original uri is replaced.
//     * Relative path, i.e.  xx?yy=abc . In this case the original RequestURI
//       is updated according to the new relative path.
func (u *URI) Update(newURI string) {
	u.UpdateBytes(s2b(newURI))
}

// UpdateBytes updates uri.
//
// The following newURI types are accepted:
//
//     * Absolute, i.e. http://foobar.com/aaa/bb?cc . In this case the original
//       uri is replaced by newURI.
//     * Absolute without scheme, i.e. //foobar.com/aaa/bb?cc. In this case
//       the original scheme is preserved.
//     * Missing host, i.e. /aaa/bb?cc . In this case only RequestURI part
//       of the original uri is replaced.
//     * Relative path, i.e.  xx?yy=abc . In this case the original RequestURI
//       is updated according to the new relative path.
func (u *URI) UpdateBytes(newURI []byte) {
	u.requestURI = u.updateBytes(newURI, u.requestURI)
}

func (u *URI) updateBytes(newURI, buf []byte) []byte {
	if len(newURI) == 0 {
		return buf
	}

	n := bytes.Index(newURI, strSlashSlash)
	if n >= 0 {
		// absolute uri
		var b [32]byte
		schemeOriginal := b[:0]
		if len(u.scheme) > 0 {
			schemeOriginal = append([]byte(nil), u.scheme...)
		}
		u.Parse(nil, newURI)
		if len(schemeOriginal) > 0 && len(u.scheme) == 0 {
			u.scheme = append(u.scheme[:0], schemeOriginal...)
		}
		return buf
	}

	if newURI[0] == '/' {
		// uri without host
		buf = u.appendSchemeHost(buf[:0])
		buf = append(buf, newURI...)
		u.Parse(nil, buf)
		return buf
	}

	// relative path
	switch newURI[0] {
	case '?':
		// query string only update
		u.SetQueryStringBytes(newURI[1:])
		return append(buf[:0], u.FullURI()...)
	case '#':
		// update only hash
		u.SetHashBytes(newURI[1:])
		return append(buf[:0], u.FullURI()...)
	default:
		// update the last path part after the slash
		path := u.Path()
		n = bytes.LastIndexByte(path, '/')
		if n < 0 {
			panic("BUG: path must contain at least one slash")
		}
		buf = u.appendSchemeHost(buf[:0])
		buf = appendQuotedPath(buf, path[:n+1])
		buf = append(buf, newURI...)
		u.Parse(nil, buf)
		return buf
	}
}

// FullURI returns full uri in the form {Scheme}://{Host}{RequestURI}#{Hash}.
func (u *URI) FullURI() []byte {
	u.fullURI = u.AppendBytes(u.fullURI[:0])
	return u.fullURI
}

// AppendBytes appends full uri to dst and returns the extended dst.
func (u *URI) AppendBytes(dst []byte) []byte {
	dst = u.appendSchemeHost(dst)
	return append(dst, u.RequestURI()...)
}

func (u *URI) appendSchemeHost(dst []byte) []byte {
	dst = append(dst, u.Scheme()...)
	dst = append(dst, strColonSlashSlash...)
	return append(dst, u.Host()...)
}

// WriteTo writes full uri to w.
//
// WriteTo implements io.WriterTo interface.
func (u *URI) WriteTo(w io.Writer) (int64, error) {
	n, err := w.Write(u.FullURI())
	return int64(n), err
}

// String returns full uri.
func (u *URI) String() string {
	return string(u.FullURI())
}

func splitHostURI(host, uri []byte) ([]byte, []byte, []byte) {
	n := bytes.Index(uri, strSlashSlash)
	if n < 0 {
		return strHTTP, host, uri
	}
	scheme := uri[:n]
	if bytes.IndexByte(scheme, '/') >= 0 {
		return strHTTP, host, uri
	}
	if len(scheme) > 0 && scheme[len(scheme)-1] == ':' {
		scheme = scheme[:len(scheme)-1]
	}
	n += len(strSlashSlash)
	uri = uri[n:]
	n = bytes.IndexByte(uri, '/')
	if n < 0 {
		// A hack for bogus urls like foobar.com?a=b without
		// slash after host.
		if n = bytes.IndexByte(uri, '?'); n >= 0 {
			return scheme, uri[:n], uri[n:]
		}
		return scheme, uri, strSlash
	}
	return scheme, uri[:n], uri[n:]
}

// QueryArgs returns query args.
func (u *URI) QueryArgs() *Args {
	u.parseQueryArgs()
	return &u.queryArgs
}

func (u *URI) parseQueryArgs() {
	if u.parsedQueryArgs {
		return
	}
	u.queryArgs.ParseBytes(u.queryString)
	u.parsedQueryArgs = true
}


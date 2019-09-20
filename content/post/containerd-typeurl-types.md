---
title: "containerd-typeurl-type"
date: 2019-09-13T11:20:18+08:00
draft: true
categories: [""]
tags: ["containerd"]
---
# 简述
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/containerd/typeurl/typeurl.go
{{< /highlight >}}

package typeurl

import (
	"github.com/gogo/protobuf/proto"
	"github.com/gogo/protobuf/types"
	"github.com/pkg/errors"
)
# 变量
{{< highlight go "linenos=inline" >}}
var (
	mu       sync.Mutex
	registry = make(map[reflect.Type]string)
)

var ErrNotFound = errors.New("not found")
{{< /highlight >}}

# Register
{{< highlight go "linenos=inline" >}}
// Register a type with a base URL for JSON marshaling. When the MarshalAny and
// UnmarshalAny functions are called they will treat the Any type value as JSON.
// To use protocol buffers for handling the Any value the proto.Register
// function should be used instead of this function.
func Register(v interface{}, args ...string) {
	var (
		t = tryDereference(v)
		p = path.Join(args...)
	)
	mu.Lock()
	defer mu.Unlock()
	if et, ok := registry[t]; ok {
		if et != p {
			panic(errors.Errorf("type registred with alternate path %q != %q", et, p))
		}
		return
	}
	registry[t] = p
}
{{< /highlight >}}
## path.Join
Join joins any number of path elements into a single path, adding a separating slash if necessary. The result is Cleaned; in particular, all empty strings are ignored.
{{< highlight go "linenos=inline" >}}
// import "path"
func Join(elem ...string) string
{{< /highlight >}}

# TypeURL
{{< highlight go "linenos=inline" >}}
// TypeURL returns the type url for a registred type.
func TypeURL(v interface{}) (string, error) {
	mu.Lock()
	u, ok := registry[tryDereference(v)]
	mu.Unlock()
	if !ok {
		// fallback to the proto registry if it is a proto message
		pb, ok := v.(proto.Message)
		if !ok {
			return "", errors.Wrapf(ErrNotFound, "type %s", reflect.TypeOf(v))
		}
		return proto.MessageName(pb), nil
	}
	return u, nil
}
{{< /highlight >}}

# Is
{{< highlight go "linenos=inline" >}}
// Is returns true if the type of the Any is the same as v.
func Is(any *types.Any, v interface{}) bool {
	// call to check that v is a pointer
	tryDereference(v)
	url, err := TypeURL(v)
	if err != nil {
		return false
	}
	return any.TypeUrl == url
}
{{< /highlight >}}

// MarshalAny marshals the value v into an any with the correct TypeUrl.
// If the provided object is already a proto.Any message, then it will be
// returned verbatim. If it is of type proto.Message, it will be marshaled as a
// protocol buffer. Otherwise, the object will be marshaled to json.
func MarshalAny(v interface{}) (*types.Any, error) {
	var marshal func(v interface{}) ([]byte, error)
	switch t := v.(type) {
	case *types.Any:
		// avoid reserializing the type if we have an any.
		return t, nil
	case proto.Message:
		marshal = func(v interface{}) ([]byte, error) {
			return proto.Marshal(t)
		}
	default:
		marshal = json.Marshal
	}

	url, err := TypeURL(v)
	if err != nil {
		return nil, err
	}

	data, err := marshal(v)
	if err != nil {
		return nil, err
	}
	return &types.Any{
		TypeUrl: url,
		Value:   data,
	}, nil
}

// UnmarshalAny unmarshals the any type into a concrete type.
func UnmarshalAny(any *types.Any) (interface{}, error) {
	return UnmarshalByTypeURL(any.TypeUrl, any.Value)
}

func UnmarshalByTypeURL(typeURL string, value []byte) (interface{}, error) {
	t, err := getTypeByUrl(typeURL)
	if err != nil {
		return nil, err
	}
	v := reflect.New(t.t).Interface()
	if t.isProto {
		err = proto.Unmarshal(value, v.(proto.Message))
	} else {
		err = json.Unmarshal(value, v)
	}
	return v, err
}

type urlType struct {
	t       reflect.Type
	isProto bool
}

func getTypeByUrl(url string) (urlType, error) {
	for t, u := range registry {
		if u == url {
			return urlType{
				t: t,
			}, nil
		}
	}
	// fallback to proto registry
	t := proto.MessageType(url)
	if t != nil {
		return urlType{
			// get the underlying Elem because proto returns a pointer to the type
			t:       t.Elem(),
			isProto: true,
		}, nil
	}
	return urlType{}, errors.Wrapf(ErrNotFound, "type with url %s", url)
}

# tryDereference
解引用，检查并获取引用的元素类型
{{< highlight go "linenos=inline" >}}
func tryDereference(v interface{}) reflect.Type {
	t := reflect.TypeOf(v)
	if t.Kind() == reflect.Ptr {
		// require check of pointer but dereference to register
		return t.Elem()
	}
	panic("v is not a pointer to a type")
}
{{< /highlight >}}
## reflect.Kind
A Kind represents the specific kind of type that a Type represents. The zero Kind is not a valid kind.
{{< highlight go "linenos=inline" >}}
type Kind uint
{{< /highlight >}}

## reflect.Type
{{< highlight go "linenos=inline" >}}
type Type interface {
    ......
    // Kind returns the specific kind of this type.
    Kind() Kind
    ......
    // Elem returns a type's element type.
    // It panics if the type's Kind is not Array, Chan, Map, Ptr, or Slice.
    Elem() Type

}
{{< /highlight >}}

## reflect.TypeOf
TypeOf returns the reflection Type that represents the dynamic type of i. If i is a nil interface value, TypeOf returns nil.
{{< highlight go "linenos=inline" >}}
func TypeOf(i interface{}) Type
{{< /highlight >}}

## reflect.Ptr
{{< highlight go "linenos=inline" >}}
const (
    ......
    Invalid Kind = iota
    Bool
    Int
    Int8
    ......
    Map
    Ptr
    Slice
    String
    Struct
    UnsafePointer
    ......
)
{{< /highlight >}}

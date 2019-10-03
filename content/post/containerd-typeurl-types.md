---
title: "containerd-typeurl-type"
date: 2019-09-13T11:20:18+08:00
draft: true
categories: ["技术"]
tags: ["containerd"]
---
# 简述
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/containerd/typeurl/typeurl.go
{{< /highlight >}}

# 变量
{{< highlight go "linenos=inline" >}}
var (
	mu       sync.Mutex
	registry = make(map[reflect.Type]string)
)

var ErrNotFound = errors.New("not found")
{{< /highlight >}}

# Register
Register用于给URL注册一个类型，以便JSON编码，当调用MarshalAny或者UnmarshalAny函数时，这些URL就可视为一个Any类型来处理了。v表类型，args部分通过path.Join后，形成一个URL路径。  
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
以下是在[github.com/containerd/containerd/client.go](http://www.zvier.top/post/containerd-containerd-client/#init)中对Register函数的调用
{{< highlight go "linenos=inline" >}}
...
typeurl.Register(&specs.Spec{}, prefix, "opencontainers/runtime-spec", major, "Spec")
...
{{< /highlight >}}
再来看下两个相关的单元测试
## 给URL注册类型
{{< highlight go "linenos=inline" >}}
go test -v types_test.go types.go -test.run TestRegisterPointerGetPointer
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
func TestRegisterPointerGetPointer(t *testing.T) {
    clear()
    expected := "test"
    Register(&test{}, "test")

    url, err := TypeURL(&test{})
    if err != nil {
        t.Fatal(err)
    }
    if url != expected {
        t.Fatalf("expected %q but received %q", expected, url)
    }
}
{{< /highlight >}}
也就是说当我们将URL路径<code>test</code>注册到一个<code>test</code>实例时，那TypeURL这个对象返回的就是该类型的URL，即<code>test</code>。
## 注册不同的URL到同一个类型将panic
{{< highlight go "linenos=inline" >}}
func TestRegisterDiffUrls(t *testing.T) {
    clear()
    defer func() {
        if err := recover(); err == nil {
            t.Error("registering the same type with different urls should panic")
        }
    }()
    Register(&test{}, "test")
    Register(&test{}, "test", "two")
}
{{< /highlight >}}

## path.Join
Join joins any number of path elements into a single path, adding a separating slash if necessary. The result is Cleaned; in particular, all empty strings are ignored.
{{< highlight go "linenos=inline" >}}
// import "path"
func Join(elem ...string) string
{{< /highlight >}}

# TypeURL
TypeURL用于返回被注册类型上注册的URL
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
Is用于判断types.Any实例和v上注册的TypeUrl是否相同
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
{{< highlight go "linenos=inline" >}}
func TestIs(t *testing.T) {
    clear()
    Register(&test{}, "test")

    v := &test{
        Name: "koye",
        Age:  6,
    }
    any, err := MarshalAny(v)
    if err != nil {
        t.Fatal(err)
    }
    if !Is(any, &test{}) {
        t.Fatal("Is(any, test{}) should be true")
    }
}
{{< /highlight >}}
可见，一个test实例通过MarshalAny编码后会生成一个Any结构的实例，该实例的TypeUrl和我们注册在test上的TypeUrl预期是一致的。  

# MarshalAny
MarshalAny用于将给定的v编码成一个具备TypeUrl值的types.Any结构，如果给定对象已经拥有proto.Any信息，那就直接将该信息完整返回，否则都需要编码处理一下。

{{< highlight go "linenos=inline" >}}
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
{{< /highlight >}}
看下单元测试，先是将URL路径test注册到了一个test实例，然后定义了一个test实例并调用MarshalAny编码生成一个Any对象，那这个对象的TypeUrl预期应该就是test，再次编码这个any对象，因为它已经属于types.Any类型，直接返回，所以预期应该相同。
{{< highlight go "linenos=inline" >}}
func TestMarshal(t *testing.T) {
    clear()
    expected := "test"
    Register(&test{}, "test")

    v := &test{
        Name: "koye",
        Age:  6,
    }
    any, err := MarshalAny(v)
    if err != nil {
        t.Fatal(err)
    }
    if any.TypeUrl != expected {
        t.Fatalf("expected %q but received %q", expected, any.TypeUrl)
    }

    // marshal it again and make sure we get the same thing back.
    newany, err := MarshalAny(any)
    if err != nil {
        t.Fatal(err)
    }

    if newany != any { // you that right: we want the same *pointer*!
        t.Fatalf("expected to get back same object: %v != %v", newany, any)
    }

}
{{< /highlight >}}

# UnmarshalAny
给定一个types.Any类型对象，UnmarshalAny将其进行解码操作，返回本真  
{{< highlight go "linenos=inline" >}}
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
{{< /highlight >}}
以下单元测试反应了将一个test实例编码，解码后，得到的结构仍然可以断言成test类型，信息原封不动  
{{< highlight go "linenos=inline" >}}
func TestMarshalUnmarshal(t *testing.T) {
    clear()
    Register(&test{}, "test")

    v := &test{
        Name: "koye",
        Age:  6,
    }
    any, err := MarshalAny(v)
    if err != nil {
        t.Fatal(err)
    }
    nv, err := UnmarshalAny(any)
    if err != nil {
        t.Fatal(err)
    }
    td, ok := nv.(*test)
    if !ok {
        t.Fatal("expected value to cast to *test")
    }
    if td.Name != "koye" {
        t.Fatal("invalid name")
    }
    if td.Age != 6 {
        t.Fatal("invalid age")
    }
}
{{< /highlight >}}

# getTypeByUrl
给定url，获取其类型  
{{< highlight go "linenos=inline" >}}
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
{{< /highlight >}}

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

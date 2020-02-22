---
title: "golang new与make"
date: 2020-01-11T11:27:08+08:00
draft: true
categories: ["技术"]
tags: ["golang"]
---
<!--more-->
# 简述
golang中，new和make均可以用于分配内存，区别在于new分配内存后只是简单将分配的内存置为对应类型的零置，而make既可以按需分配内存又可以初始化内存。

# new
{{< highlight go "linenos=inline" >}}
// golang/src/builtin/builtin.go
// The new built-in function allocates memory. The first argument is a type,
// not a value, and the value returned is a pointer to a newly
// allocated zero value of that type.
func new(Type) *Type

// Type is here for the purposes of documentation only. It is a stand-in
// for any Go type, but represents the same type for any given function
// invocation.
type Type int
{{< /highlight >}}
可见，new的参数是一个类型，返回值为指向该类型内存地址的指针，同时会把分配的内存置为类型对应的零零值, 比如字符为空，整型为0, 逻辑值为false。如果Type为slice，map，chan等引用类型，则初始化为nil，值为nil的变量是不能直接被赋值的，还需要使用make来分配并初始化内存。

# make
{{< highlight go "linenos=inline" >}}
// golang/src/builtin/builtin.go
// The make built-in function allocates and initializes an object of type
// slice, map, or chan (only). Like new, the first argument is a type, not a
// value. Unlike new, make's return type is the same as the type of its
// argument, not a pointer to it. The specification of the result depends on
// the type:
//  Slice: The size specifies the length. The capacity of the slice is
//  equal to its length. A second integer argument may be provided to
//  specify a different capacity; it must be no smaller than the
//  length. For example, make([]int, 0, 10) allocates an underlying array
//  of size 10 and returns a slice of length 0 and capacity 10 that is
//  backed by this underlying array.
//  Map: An empty map is allocated with enough space to hold the
//  specified number of elements. The size may be omitted, in which case
//  a small starting size is allocated.
//  Channel: The channel's buffer is initialized with the specified
//  buffer capacity. If zero, or the size is omitted, the channel is
//  unbuffered.
func make(t Type, size ...IntegerType) Type
{{< /highlight >}}
make返回的就是类型本身，而不是指针类型，因为make只能给slice，map，channel等初始化内存，它们返回的就是引用类型，就没必要返回指针了

# 总结

1. new可分配任意类型的数据，make仅能用于分配和初始化类型为slice、map、chan等类型的变量。
2. new返回的是指针类型*Type，make返回引用Type。
3. new分配的空间被清零, make分配空间后，会进行初始化。

# 参考
[golang中,new和make区别](https://zhuanlan.zhihu.com/p/97854299)

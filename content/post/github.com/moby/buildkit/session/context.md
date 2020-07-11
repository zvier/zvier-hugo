---
title: "moby buildkit session context"
date: 2020-02-10T08:05:27+08:00
draft: true
categories: ["技术"]
tags: ["buildkit"]
---
山中无甲子，寒尽不知年。——吴承恩《西游记》
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
// github.com/moby/buildkit/session/context.go
{{< /highlight >}}

# contextKeyT
{{< highlight go "linenos=inline" >}}
type contextKeyT string

var contextKey = contextKeyT("buildkit/session-id")
{{< /highlight >}}

# NewContext
{{< highlight go "linenos=inline" >}}
func NewContext(ctx context.Context, id string) context.Context {
	if id != "" {
		return context.WithValue(ctx, contextKey, id)
	}
	return ctx
}
{{< /highlight >}}

## context.WithValue
WithValue returns a copy of parent in which the value associated with key is val. WithValue returns a copy of parent in which the value associated with key is val. Use context Values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions.

The provided key must be comparable and should not be of type string or any other built-in type to avoid collisions between packages using context. Users of WithValue should define their own types for keys. To avoid allocating when assigning to an interface{}, context keys often have concrete type struct{}. Alternatively, exported context key variables' static type should be a pointer or interface.
{{< highlight go "linenos=inline" >}}
// import "context"
func WithValue(parent Context, key, val interface{}) Context
{{< /highlight >}}

# FromContext
{{< highlight go "linenos=inline" >}}
func FromContext(ctx context.Context) string {
	v := ctx.Value(contextKey)
	if v == nil {
		return ""
	}
	return v.(string)
}
{{< /highlight >}}

## Context

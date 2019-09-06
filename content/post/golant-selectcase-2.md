---
title: "golang reflect.Select使用解读"
date: 2019-06-24T21:32:45+08:00
draft: true
categories: ["golang"]
tags: [""]
---

# 背景
golang中的for-select模式常用于监听确定通道集合，有些时候，我们要监听的通道不是事先已知的，而是根据当前环境需要动态生成，对于监听这种不确定的通道集，就需要借助reflect.Select了。

# reflect.Select介绍
先来看一下reflect.Select函数的签名  
{{< highlight go "linenos=inline" >}}
func Select(cases []SelectCase)(chosen int, recv Value, recvOK bool)
{{< /highlight >}}

> Select executes a select operation described by the list of cases. Like the Go select statement, it blocks until at least one of the cases can proceed, makes a uniform pseudo-random choice, and then executes that case.  

> It returns the index of the chosen case and, if that case was a receive operation, the value received and a boolean indicating whether the value corresponds to a send on the channel (as opposed to a zero value received because the channel is closed).

参数cases表示我们要监听的通道集合，它是一个SelectCase实例构成的切片，每一个SelectCase绑定了待监听的通道，以及指定了通道方向，也可能包含待发送值，一睹SelectCase的声明便知   
{{< highlight go "linenos=inline" >}}
type SelectCase struct {
    Dir  SelectDir // direction of case
    Chan Value     // channel to use (for send or receive)
    Send Value     // value to send (for send)
}
{{< /highlight >}}

> A SelectCase describes a single case in a select operation. The kind of case depends on Dir, the communication direction(通信方向).

If Dir is SelectDefault, the case represents a default case. Chan and Send must be zero Values.

If Dir is SelectSend, the case represents a send operation. Normally Chan's underlying value must be a channel(Chan字段底层必须是一个chan类型), and Send's underlying value must be assignable to the channel's element type. As a special case, if Chan is a zero Value, then the case is ignored, and the field Send will also be ignored and may be either zero or non-zero.

If Dir is SelectRecv, the case represents a receive operation. Normally Chan's underlying value must be a channel and Send must be a zero Value. If Chan is a zero Value, then the case is ignored, but Send must still be a zero Value. When a receive operation is selected, the received Value is returned by Select.

# reflect.Select的使用
{{< highlight go "linenos=inline" >}}
cases := make([]reflect.SelectCase, len(chans))
for i, ch := range chans {
    cases[i] = reflect.SelectCase{
        Dir: reflect.SelectRecv,
        Chan: reflect.ValueOf(ch),
    }
}
// ok will be true if the channel has not been closed.
chosen ,value, ok := reflect.Select(cases)
msg := value.String()
{{< /highlight >}}
更常规的，我们是结合for一起使用，实现对不确定通道的监听，直到所有通道到关闭。  
{{< highlight go "linenos=inline" >}}
remaining := len(cases)
for remaining > 0 {
	chosen, value, ok := reflect.Select(cases)
	if !ok {
		// The chosen channel has been closed, so zero out the channel to disable the case
		cases[chosen].Chan = reflect.ValueOf(nil)
		remaining -= 1
		continue
	}

	fmt.Printf("Read from channel %#v and received %s\n", chans[chosen], value.String())
}
{{< /highlight >}}
需要注意的是，与linux中的select一样，reflect.Select或select是水平触发，如果Select的通道已经关闭，那ok值就是false，这时我们应该把该SelectCase的Chan字段置nil或挪出监听通道集，避免下次又别选中，同时改变计数器以便在合适的时机跳出for循环，下面这种方式也是可行的
 {{< highlight go "linenos=inline" >}}
     // Select loop
    for len(cases) > 0 {
        // that `v` is the only value "in-flight" that any of
        // the workers have sent. All other workers are blocked
        // trying to send the single value they have calculated
        // where-as the goroutine version reads/buffers a single
        // extra value from each worker.
        i, v, ok := reflect.Select(cases)
        if !ok {
            // Channel cases[i] has been closed, remove it
            // from our slice of cases and update our ids
            // mapping as well.
            cases = append(cases[:i], cases[i+1:]...)
            ids = append(ids[:i], ids[i+1:]...)
            continue
        }

        // Process each value
        fn(ids[i], v.String())
    }
{{< /highlight >}}

# 参考
[how to listen to N channels? (dynamic select statement)](https://stackoverflow.com/questions/19992334/how-to-listen-to-n-channels-dynamic-select-statement)
[The Go Playground](https://play.golang.org/p/8zwvSk4kjx)



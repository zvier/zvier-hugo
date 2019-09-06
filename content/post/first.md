---
title: "golang interface（一）"
date: 2019-06-02T21:19:33+08:00
tags: ["golang"]
categories: ["技术", "编程语言"]
draft: false
---
* [什么是鸭子类型](#什么是鸭子类型)
* [什么是interface](#什么是interface)
* [golang中的interface](#golang中的interface)
* [总结](#总结)
* [参考](#参考)


接口(interface)是golang中一个非常重要的概念，通过灵活的接口，我们可以实现很多面向对象的特性。
<!--more-->

<!-- vim-markdown-toc GFM -->

* [什么是鸭子类型](#什么是鸭子类型)
* [什么是interface](#什么是interface)
* [golang中的interface](#golang中的interface)
* [总结](#总结)
* [参考](#参考)

<!-- vim-markdown-toc -->
# 什么是鸭子类型

> If it looks like a duck, swims like a duck, and quacks like a duck, then it probably is a duck.

如果某个东西长得像鸭子，游泳像鸭子，叫起来像鸭子，那它就是一只鸭子。

![duck](/img/go-interface/go-duck.jpg)


在动态编程语言中，鸭子类型(Duck Typing)作为一种对象推断策略，关注的是对象的行为而非对象类型本身，golang作为一门静态语言，通过接口的方式完美地支持了鸭子类型，为程序设计引入了动态编程语言中的便利。

动态语言python中，如下定义一个hell_world函数
{{< highlight python "linenos=inline" >}}
def hell_world(animal):
    animal.say_hello()
{{< /highlight >}}
当调用hell_world函数时，我们可以传入任意实现了say_hello()方法的类型实例，类似的，在golang中通过接口也可以做到这种效果
{{< highlight go "linenos=inline" >}}
type Animal interface {
    func sayHello()
}
func HellWorld(animal Animal) {
    animal.sayHello()
}
{{< /highlight >}}
而在其它传统静态编程语言中，比如C++/Java等，则必须显示地声明实现某个接口，结合动态类型绑定，才能在任何需要这个接口的地方使用具体类型。

# 什么是interface
> In object-oriented programming, **a protocol or interface is a common means for unrelated objects to communicate with each other**. These are definitions of methods and values which the objects agree upon in order to co-operate.

协议(protocol)是通信双方的一种约定，比如TCP/IP协议，HTTP协议，蓝牙协议，WiFi协议等，满足协议的双方不论具体实现，接入即可正常通信。  
![protocol](/img/go-interface/go-protocol.jpeg)

interface也可以类比协议，通过声明一系列方法对类型的行为做出了约束，这些方法不包含具体的实现，所以说它是抽象的(abstract type)。而对应的，像int，string等称之为具体类型(concrete type)。  

# golang中的interface
golang作为一门现代静态语言，一方面具备静态语言的类型检查机制，另一方面，引入了不少动态语言的便利，为了实现接口类型，golang采用来比较折中的办法，不要求类型显示声明实现某个接口，只要实现接口约定的相关方法，编译器就能检测识别到。

具体来说在golang中，interface是一组method的集合，它不关系属性(数据)，只关心行为(方法)，在实际使用中，我们只需要定义自己的类型，并实现interface里约定的所有方法，该类型就隐式实现了interface，它就可以当作该接口类型来使用。  

> 如果一个具体类型要实现某个interface，它必须实现该interface约定的所有方法。

io包中就有关于Writer和Closer的interface示例：  
{{< highlight go "linenos=inline" >}}
type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
func Fprintf(w io.Writer, format string, args ...interface{}) (int, error)

func Printf(format string, args ...interface{}) (int, error) {
    return Fprintf(os.Stdout, format, args...)
}

func Sprintf(format string, args ...interface{}) string {
    var buf bytes.Buffer
    Fprintf(&buf, format, args...)
    return buf.String()
}
{{< /highlight >}}

这里，buf的类型为bytes.Buffer，os.Stdout的类型为*os.File，具体类型不同，但由于它们都实现了os.Writer这个interface，所以都能被Fprintf使用，也就是说Fprintf函数并不关心w参数的具体类型，只要w实现了interface所约定的Write method，即可通通放心使用。  

这正是interface的特点，只在乎其行为，不在乎属性，让我们可以自由地向函数传递任何满足接口类型的concrete type。

另外，golang中允许有不带任何方法的interface，不带任何方法的interface叫做empty interface。可以推断，所有类型都实现了empty interface。

# 总结
1. 如果一个具体类型实现了某个interface，它必须实现该interface约定的所有方法
2. interface只在乎其行为，不在乎其值
3. golang中，所有类型都实现了empty interface

# 参考
[Go interface 详解(一) ：介绍](https://sanyuesha.com/2017/10/10/go-interface-1/)

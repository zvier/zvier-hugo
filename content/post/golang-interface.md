---
title: "golang interface"
date: 2019-06-02T21:19:33+08:00
tags: ["golang"]
categories: ["技术"]
tags: ["golang"]
draft: false
---
丙辰中秋，欢饮达旦，大醉，作此篇，兼怀子由。  
明月几时有？把酒问青天。不知天上宫阙，今夕是何年。我欲乘风归去，又恐琼楼玉宇，高处不胜寒。起舞弄清影，何似在人间。  
转朱阁，低绮户，照无眠。不应有恨，何事长向别时圆？人有悲欢离合，月有阴晴圆缺，此事古难全。但愿人长久，千里共婵娟。
但愿人长久，千里共婵娟。——苏轼《水调歌头·丙辰中秋》

<!--more-->
# 简述
接口(interface)是golang中一个非常重要和常用的语法，通过灵活的接口，我们可以实现很多面向对象的特性。

<!-- vim-markdown-toc -->
# 什么是鸭子类型

> If it looks like a duck, swims like a duck, and quacks like a duck, then it probably is a duck.

如果某个东西长得像鸭子，游泳像鸭子，叫起来像鸭子，那它就是一只鸭子。

![duck](/img/go-interface/go-duck.jpg)


在动态编程语言中，鸭子类型(Duck Typing)作为一种对象推断策略，关注的是<font color="red">**对象的行为**</font>，而非对象类型本身，golang作为一门静态语言，通过接口的方式完美地支持了鸭子类型，为程序设计引入了动态编程语言中的便利。

先来看一下动态语言python中的鸭子类型，如下定义一个hell_world函数，当调用hell_world函数时，我们可以传入任意实现了say_hello()方法的类型实例。  
{{< highlight python "linenos=inline" >}}
def hell_world(animal):
    animal.say_hello()
{{< /highlight >}}

类似的，在golang中通过接口也可以做到这种效果
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

具体来说在golang中，interface是一组method的集合，它<font color="red">**不关心属性(数据)，只关心行为(方法)**</font>，在实际使用中，我们只需要定义自己的类型，并实现interface里约定的所有方法，该类型就隐式实现了interface，它就可以当作该接口类型来使用。  

> 如果一个具体类型要实现某个interface，它必须实现该interface约定的所有方法。

# 示例
io包中就有关于Writer和Closer的interface示例：  
{{< highlight go "linenos=inline" >}}
type Writer interface {
    Write(p []byte) (n int, err error)
}
{{< /highlight >}}
Writer要具备Write方法，它有向byte切片中写数据，并返回写入字节数和错误信息的功能

{{< highlight go "linenos=inline" >}}
type Closer interface {
    Close() error
}
{{< /highlight >}}
Closer要具备Close方法，它有关闭并返回错误信息的功能


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

# interface实现原理
golang中有两种interface，iface和eface
## iface
iface是带有方法的interface，结构如下:
{{< highlight go "linenos=inline" >}}
// golang/src/runtime/runtime2.go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}
{{< /highlight >}}
其中tab类似与C++中的vptr，包含了对应的方法数组和实现该接口的类型元数据  
data是实现该接口的类型的实例指针

## eface
不带方法的interface即empty interface，结构如下:
{{< highlight go "linenos=inline" >}}
// golang/src/runtime/runtime2.go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
{{< /highlight >}}

# itab
再来看下itab的结构
{{< highlight go "linenos=inline" >}}
// golang/src/runtime/runtime2.go
// layout of Itab known to compilers
// allocated in non-garbage-collected memory
// Needs to be in sync with
// ../cmd/compile/internal/gc/reflect.go:/^func.dumptypestructs.
type itab struct {
    inter *interfacetype
    _type *_type
    hash  uint32 // copy of _type.hash. Used for type switches.
    _     [4]byte
    fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
{{< /highlight >}}
inter：表示这个interface value所属的接口元信息  
_type：表示具体实现类型的元信息   
fun：表示该interface的方法数组   

# interfacetype
interfacetype定义interface 的一种抽象，包括pkg path，method
{{< highlight go "linenos=inline" >}}
// golang/src/runtime/type.go
type interfacetype struct {
    typ     _type
    pkgpath name
    mhdr    []imethod
}
type imethod struct {
    name nameOff
    ityp typeOff
}
{{< /highlight >}}

# _type
_type用于保存数据类型的基本信息，数据类型占用的内存大小（size字段），数据类型的名称（nameOff字段）
{{< highlight go "linenos=inline" >}}
// golang/src/runtime/type.go
// Needs to be in sync with ../cmd/link/internal/ld/decodesym.go:/^func.commonsize,
// ../cmd/compile/internal/gc/reflect.go:/^func.dcommontype and
// ../reflect/type.go:/^type.rtype.
type _type struct {
    size       uintptr
    ptrdata    uintptr // size of memory prefix holding all pointers
    hash       uint32
    tflag      tflag
    align      uint8
    fieldalign uint8
    kind       uint8
    alg        *typeAlg
    // gcdata stores the GC type data for the garbage collector.
    // If the KindGCProg bit is set in kind, gcdata is a GC program.
    // Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
    gcdata    *byte
    str       nameOff
    ptrToThis typeOff
}
{{< /highlight >}}

# 类型转换
## 无方法interface类型转换
无方法interface类型转换将调用convT2EXXX系列函数实现，convT2EXXX函数中会调用mallocgc函数，负责给eface结构中的data字段填充
{{< highlight go "linenos=inline" >}}
// golang/src/runtime/iface.go
// The conv and assert functions below do very similar things.
// The convXXX functions are guaranteed by the compiler to succeed.
// The assertXXX functions may fail (either panicking or returning false,
// depending on whether they are 1-result or 2-result).
// The convXXX functions succeed on a nil input, whereas the assertXXX
// functions fail on a nil input.

func convT2E(t *_type, elem unsafe.Pointer) (e eface) {
    if raceenabled {
        raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2E))
    }
    if msanenabled {
        msanread(elem, t.size)
    }
    x := mallocgc(t.size, t, true)
    // TODO: We allocate a zeroed object only to overwrite it with actual data.
    // Figure out how to avoid zeroing. Also below in convT2Eslice, convT2I, convT2Islice.
    typedmemmove(t, x, elem)
    e._type = t
    e.data = x
    return
}

{{< /highlight >}}

对于有method的interface，则调用形如convT2IXXX实现
{{< highlight python "linenos=inline" >}}
func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
    t := tab._type
    if raceenabled {
        raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2I))
    }
    if msanenabled {
        msanread(elem, t.size)
    }
    x := mallocgc(t.size, t, true)
    typedmemmove(t, x, elem)
    i.tab = tab
    i.data = x
    return
}

func convT2Inoptr(tab *itab, elem unsafe.Pointer) (i iface) {
    t := tab._type
    if raceenabled {
        raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2Inoptr))
    }
    if msanenabled {
        msanread(elem, t.size)
    }
    x := mallocgc(t.size, t, false)
    memmove(x, elem, t.size)
    i.tab = tab
    i.data = x
    return
}

func convT16(val uint16) (x unsafe.Pointer) {
    if val == 0 {
        x = unsafe.Pointer(&zeroVal[0])
    } else {
        x = mallocgc(2, uint16Type, false)
        *(*uint16)(x) = val
    }
    return
}
{{< /highlight >}}

{{< highlight python "linenos=inline" >}}
func convT32(val uint32) (x unsafe.Pointer)
func convT64(val uint64) (x unsafe.Pointer)
func convTstring(val string) (x unsafe.Pointer)
func convTslice(val []byte) (x unsafe.Pointer)
func convT2Enoptr(t *_type, elem unsafe.Pointer) (e eface)
{{< /highlight >}}

对于断言的实现则为assertXXX实现函数
{{< highlight python "linenos=inline" >}}
func assertI2I(inter *interfacetype, i iface) (r iface) {
    tab := i.tab
    if tab == nil {
        // explicit conversions require non-nil interface value.
        panic(&TypeAssertionError{nil, nil, &inter.typ, ""})
    }
    if tab.inter == inter {
        r.tab = tab
        r.data = i.data
        return
    }
    r.tab = getitab(inter, tab._type, false)
    r.data = i.data
    return
}

func assertI2I2(inter *interfacetype, i iface) (r iface, b bool) {
    tab := i.tab
    if tab == nil {
        return
    }
    if tab.inter != inter {
        tab = getitab(inter, tab._type, true)
        if tab == nil {
            return
        }
    }
    r.tab = tab
    r.data = i.data
    b = true
    return
}
{{< /highlight >}}



# 总结
1. 如果一个具体类型实现了某个interface，它必须实现该interface约定的所有方法
2. interface只在乎其行为，不在乎其值
3. golang中，所有类型都实现了empty interface

# 参考
[Go interface 详解(一) ：介绍](https://sanyuesha.com/2017/10/10/go-interface-1/)

---
title: "golang 汇编"
date: 2019-07-12T21:36:38+08:00
draft: true
categories: ["golang"]
tags: [""]
---

# golang汇编是什么
golang汇编源于plan9，它不是一门独立的语言，无法单独使用，必须以golang包的方式提供才能使用，同时包中至少要有一个golang语言文件，用来指明当前包的包名，以及导出汇编文件中的变量名、函数名等符号信息，以便汇编代码中定义的变量和函数可以被其它golang代码引用，这时用于变量和函数定义golang汇编文件类似于c语言中的.c文件，而用于导出汇编中定义符号的golang语言源文件类似于C语言的.h文件。  

# 为什么需要golang汇编
在一些特殊场景下，我们代码实现需要做一些特殊的优化，比如：  

* 算法加速。golang编译器生成的机器码基本上都是通用代码，优化程度一般，在一些特殊场景下，需要用到特殊的优化逻辑或CPU指令来加速算法。   
* 摆脱golang编译器的约束。比如通过汇编来调用其它package的私有函数。    
* 进行一些hack的事。比如通过汇编适配其他语言的ABI来直接调用其他语言的函数。   
* 利用//go:noescape进行内存分配优化，golang编译器的逃逸分析机制，可以用于决定当前变量是分配在堆内存上还是栈上。但是有时逃逸分析的结果并不让人满意，一些变量完全可以分配在栈上，但是逃逸分析将其移动到堆上，因此我们需要使用golang编译器的go:noescape将其转换，强制分配在函数栈上。当然也可以强制让对象分配在堆上。

# 如何使用
对于如何使用golang汇编，先用个例子入个门  
{{< highlight go "linenos=inline" >}}
package pkg

var NUM = 2019
{{< /highlight >}}

上面代码中定义了一个int类型的NUM变量并初始化为2019，使用如下命令可查看对应的伪汇编代码  

{{< highlight bash "linenos=inline" >}}
go tool compile -S test.go
"".NUM SNOPTRDATA size=8
	0x0000 e3 07 00 00 00 00 00 00                          ........
{{< /highlight >}}

> go tool compile命令用于调用go语言提供的底层命令工具，-S参数表示输出汇编格式代码  

上面汇编输出结果中:

* "".NUM对应NUM变量的符号  
* SNOPTRDATA是一个标志，NOPTR表示数据中不包含指针数据  
* size=8表这个变量占用内存大小为8个字节  
* 0x0000
* e3 07 00 00 00 00 00 00是一串小端表示的十六进制数字，共8个字节，正常大端可读为0x07e3，表变量的初始化值，对应十进制中的2019

## golang汇编DATA命令
golang汇编语言提供了一个DATA命令用于初始化包变量，DATA命令的语法如下：
{{< highlight bash "linenos=inline" >}}
DATA .symbol+offset(SB)/width, value
{{< /highlight >}}

其中.symbol为golang语言代码中声明的符号symbol在汇编语言中对应的标识符，点符号为一个特殊的unicode符号，offset是符号开始地址的偏移量，width是要初始化内存的宽度大小，value是要初始化的值，如果是常量，value值需要以美元符号$开头。

我们采用以下命令可以给NUM变量初始化为十六进制的0x07e3，对应十进制的2019：
{{< highlight bash "linenos=inline" >}}
DATA ·NUM+0(SB)/1,$0xe3
DATA ·NUM+1(SB)/1,$0x07
{{< /highlight >}}

## golang汇编GLOBL命令
变量定义好之后需要导出以供其它代码引用。go汇编语言提供了GLOBL命令用于将符号导出：
{{< highlight bash "linenos=inline" >}}
GLOBL symbol(SB), width
{{< /highlight >}}

其中symbol对应汇编中符号的名字，width为符号对应内存的大小。用以下命令将汇编中的·NUM变量导出：

{{< highlight bash "linenos=inline" >}}
GLOBL ·NUM, $8
{{< /highlight >}}

## 引用
现在已经初步完成了用汇编定义一个整数变量的工作。
为了便于其它包使用该NUM变量，我们还需要在Go代码中声明该变量，同时也给变量指定一个合适的类型。修改pkg.go的内容如下：
{{< highlight go "linenos=inline" >}}
package pkg
var NUM int
{{< /highlight >}}
现在go语言的代码不再是定义一个变量，语义变成了声明一个变量（声明一个变量时不能再进行初始化操作）。而NUM变量的定义工作已经在汇编语言中完成了。

将完整的汇编代码放到pkg_amd64.s(具体文件名不在意，后缀必须是.s)文件中，并与pkg.go代码置于同一目录下：
{{< highlight go "linenos=inline" >}}
GLOBL ·NUM(SB),$8

DATA ·NUM+0(SB)/1,$0xe3
DATA ·NUM+1(SB)/1,$0x07
DATA ·NUM+2(SB)/1,$0x00
DATA ·NUM+3(SB)/1,$0x00
DATA ·NUM+4(SB)/1,$0x00
DATA ·NUM+5(SB)/1,$0x00
DATA ·NUM+6(SB)/1,$0x00
DATA ·NUM+7(SB)/1,$0x00
{{< /highlight >}}
文件名pkg_amd64.s的后缀名表示AMD64环境下的汇编代码文件。

{{< highlight go "linenos=inline" >}}
tree test
test
├── pkg
│   ├── pkg_amd64.s
│   └── pkg.go
└── test.go
{{< /highlight >}}

虽然pkg包是用汇编实现，但是用法和之前的go语言版本完全一样：
{{< highlight go "linenos=inline" >}}
package main

import (
	"fmt"
	"test/pkg"
)

func main() {
	fmt.Println(pkg.NUM)
}
{{< /highlight >}}
对于go包使用者来说，用go语言或go汇编语言实现并无任何区别。

# 定义字符串变量
从go语言角度看，定义字符串和整数变量的写法基本相同，但字符串底层却有着比单个整数更复杂的数据结构。
{{< highlight go "linenos=inline" >}}
package pkg

var Name = "gopher"
{{< /highlight >}}

同样查看下pkg.go的汇编输出  
{{< highlight go "linenos=inline" >}}
go tool compile -S pkg.go
go.string."gopher" SRODATA dupok size=6
	0x0000 67 6f 70 68 65 72                                gopher
"".Name SDATA size=16
	0x0000 00 00 00 00 00 00 00 00 06 00 00 00 00 00 00 00  ................
	rel 0+8 t=1 go.string."gopher"+0
{{< /highlight >}}

汇编输出中有一个新的符号go.string."gopher"，从其长度和内容可知对应底层的"gopher"字符串数据。因为go语言的字符串并不是值类型，而只是一个只读的引用类型。如果多个代码中出现了相同的"gopher"只读字符串时，程序链接后可以引用到同一个符号go.string."gopher"。因此，该符号有一个SRODATA标志，表示这个数据是在只读内存段，dupok表示出现多个相同标识符的数据时只保留一个就可以了。

> SNOPTRDATA标志，表示数据中不包含指针数据；SRODATA标志，表示数据在只读内存段


而真正的Go字符串变量Name对应的大小却只有16个字节了。其实Name变量并没有直接对应“gopher”字符串，而是对应16字节大小的reflect.StringHeader结构体：
type reflect.StringHeader struct {
    Data uintptr
        Len  int
}
从汇编角度看，Name变量其实对应的是reflect.StringHeader结构体类型。前8个字节对应底层真实字符串数据的指针，也就是符号go.string."gopher"对应的地址。后8个字节对应底层真实字符串数据的有效长度，这里是6个字节。

现在创建pkg_amd64.s文件，尝试通过汇编代码重新定义并初始化Name字符串：

GLOBL ·NameData(SB),$8
DATA  ·NameData(SB)/8,$"gopher"

GLOBL ·Name(SB),$16
DATA  ·Name+0(SB)/8,$·NameData(SB)
DATA  ·Name+8(SB)/8,$6
因为在Go汇编语言中，go.string."gopher"不是一个合法的符号，因此我们无法通过手工创建（这是给编译器保留的部分特权，因为手工创建类似符号可能打破编译器输出代码的某些规则）。因此我们新创建了一个·NameData符号表示底层的字符串数据。然后定义·Name符号内存大小为16字节，其中前8个字节用·NameData符号对应的地址初始化，后8个字节为常量6表示字符串长度。

当用汇编定义好字符串变量并导出之后，还需要在Go语言中声明该字符串变量。然后就可以用Go语言代码测试Name变量了：

package main

import pkg "path/to/pkg"

func main() {
    println(pkg.Name)
}
不幸的是这次运行产生了以下错误：

pkgpath.NameData: missing Go //type information for global symbol: size 8
错误提示汇编中定义的NameData符号没有类型信息。其实Go汇编语言中定义的数据并没有所谓的类型，每个符号只不过是对应一块内存而已，因此NameData符号也是没有类型的。但是Go语言是再带垃圾回收器的语言，而Go汇编语言是工作在自动垃圾回收体系框架内的。档Go语言的垃圾回收器在扫描到NameData变量的时候，无法知晓该变量内部是否包含指针，因此就出现了这种错误。错误的根本原因并不是NameData没有类型，而是NameData变量没有标注是否会含有指针信息。

通过给NameData变量增加一个NOPTR标志，表示其中不会包含指针数据可以修复该错误：

#include "textflag.h"

GLOBL ·NameData(SB),NOPTR,$8
通过给·NameData增加NOPTR标志档方式表示其中不含指针数据。我们也可以通过给·NameData变量在Go语言中增加一个不含指针并且大小为8个字节的类型来修改该错误：

package pkg

var NameData [8]byte
var Name string
我们将NameData声明为长度为8的字节数组。编译器可以通过类型分析出该变量不会包含指针，因此汇编代码中可以省略NOPTR标志。现在垃圾回收器在遇到该变量的时候就会停止内部数据的扫描。

在这个实现中，Name字符串底层其实引用的是NameData内存对应的“gopher”字符串数据。因此，如果NameData发生变化，Name字符串的数据也会跟着变化。

func main() {
    println(pkg.Name)

    pkg.NameData[0] = '?'
    println(pkg.Name)
}
当然这和字符串的只读定义是冲突的，正常的代码需要避免出现这种情况。最好的方法是不要导出内部的NameData变量，这样可以避免内部数据被无意破坏。

在用汇编定义字符串时我们可以换一种思维：将底层的字符串数据和字符串头结构体定义在一起，这样可以避免引入NameData符号：

GLOBL ·Name(SB),$24

DATA ·Name+0(SB)/8,$·Name+16(SB)
DATA ·Name+8(SB)/8,$6
DATA ·Name+16(SB)/8,$"gopher"
在新的结构中，Name符号对应的内存从16字节变为24字节，多出的8个字节存放底层的“gopher”字符串。·Name符号前16个字节依然对应reflect.StringHeader结构体：Data部分对应$·Name+16(SB)，表示数据的地址为Name符号往后偏移16个字节的位置；Len部分依然对应6个字节的长度。这是C语言程序员经常使用档技巧。

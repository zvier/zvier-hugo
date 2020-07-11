---
title: "Go C Go"
date: 2020-06-25T12:05:33+08:00
draft: true
categories: [""]
tags: [""]
---
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

# golang中调用C代码
C代码可以写入到go文件中，要写在最前面，C代码用注释的写法，最后import C，例如:
{{< highlight go "linenos=inline" >}}
package main

// int add(int a, int b) {
//     return a + b;
// }
import "C"

import "fmt"

func main() {
    a := C.int(1)
    b := C.int(2)
    value := C.add(a, b)
    fmt.Printf("%v\n", value)
    fmt.Printf("%v\n", int(value))
}
{{< /highlight >}}


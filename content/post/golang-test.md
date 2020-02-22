---
title: "golang test"
date: 2020-02-21T20:51:12+08:00
draft: true
categories: ["技术"]
tags: ["golang"]
---
况是青春日将暮，桃花乱落如红雨。——唐.李贺《将进酒》
<!--more-->
# 单元测试
单元测试(unit testing)，是指对软件中的最小可测试单元进行检查和验证。对于单元测试中单元的含义，如C语言中的一个函数，Java里的一个类，图形化的软件中可以指一个窗口或一个菜单等。总的来说，单元就是人为规定的最小的被测功能模块。

单元测试是在软件开发过程中要进行的最低级别的测试活动，软件的独立单元将在与程序的其他部分相隔离的情况下进行测试。

# golang测试
testing包提供了三个测试姿势: 功能测试，压力测试，代码覆盖率测试

# go test
执行<code>go test</code>会自动读取当前目录下所有名为*_test.go的文件，生产测试用的可执行文件并运行，下面结合containerd的代码详解go test的使用
{{< highlight go "linenos=inline" >}}
cd github.com/containerd/containerd/oci
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
go test
PASS
ok  	github.com/containerd/containerd/oci	0.020s
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
go test -v
=== RUN   TestReplaceOrAppendEnvValues
=== PAUSE TestReplaceOrAppendEnvValues
=== RUN   TestWithDefaultSpecForPlatform
=== PAUSE TestWithDefaultSpecForPlatform
=== RUN   TestWithDefaultPathEnv
......
{{< /highlight >}}

值得注意的是: go的test不会保证多个TestXxx是顺序执行的，但通常会按顺序执行, 为了确保顺序执行，可以采用t.Run(name string, f func)

go test支持可选参数，主要有如下:   
1. -bench regexp  执行regexp匹配的benchmarks，例如: -bench=.   
2. -cover 开启测试覆盖率  
3. -run regexp 执行regexp匹配的函数，例如: -run=Array 将执行包含有Array开头的函数   
4. -v 显示测试详细命令，默认只会显示错误测试用例信息   

# 指定单元测试文件
测试某个文件使用-file参数，go test –file xxx_test.go ，这里-file不是必须的，通常省略
{{< highlight go "linenos=inline" >}}
cd github.com/containerd/containerd/oci
go test -v spec_test.go spec.go client.go spec_opts.go  spec_opts_linux.go
{{< /highlight >}}
go test后指定测试文件，表示仅测试该文件里的所有用例，后面必要时，还需要补充该测试文件内容所依赖的其它代码文件。

{{< highlight go "linenos=inline" >}}
=== RUN   TestGenerateSpec
=== PAUSE TestGenerateSpec
=== RUN   TestGenerateSpecWithPlatform
=== PAUSE TestGenerateSpecWithPlatform
=== RUN   TestSpecWithTTY
=== PAUSE TestSpecWithTTY
......
PASS
ok  	command-line-arguments	0.014s
{{< /highlight >}}
最后一行ok 表示测试通过，command-line-arguments 是测试用例需要用到的一个包名，0.014s表示测试花费的时间

# 指定测试用例
go test x_test.go默认会执行文件内所有的测试用例，使用-run参数可以指定测试用例单独执行
{{< highlight go "linenos=inline" >}}
go test -v -run TestDevShmSize spec_opts_test.go client.go spec.go spec_opts.go  spec_opts_linux.go
=== RUN   TestDevShmSize
=== PAUSE TestDevShmSize
=== CONT  TestDevShmSize
--- PASS: TestDevShmSize (0.00s)
PASS
ok  	command-line-arguments	0.005s
{{< /highlight >}}
注意这里是基于正则匹配执行，匹配到的用例都会被执行


# 编写单元测试  
编写单元测试时，要注意如下要点:  
1. 测试文件命名以_test.go结尾   
2. 每一个测试文件都必须import testing  
2. 测试文件的package与源码包名一致
3. 编写测试用例函数时，功能测试用例必须以Test为前缀，参数testing.T，压力测试用例必须以Benchmark为前缀，参数testing.B，形如:   
{{< highlight go "linenos=inline" >}}
func TestXXX( t *testing.T )
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/oci/spec_test.go
func TestPopulateDefaultUnixSpec(t *testing.T) {
    var (
        c   = containers.Container{ID: "TestWithDefaultSpec"}
        ctx = namespaces.WithNamespace(context.Background(), "test")
    )
    var expected Spec

    populateDefaultUnixSpec(ctx, &expected, c.ID)
    if expected.Linux == nil {
        t.Error("Cannot populate Unix Spec")
    }
}
{{< /highlight >}}
4. SkipNow()为跳过test，并直接按PASS处理下一个test，且必须写在test case的第一行, 否则无效

测试用例文件不会参与正常源码编译，也不会被包含到可执行文件中，也不需要main()作为函数执行入口，文件中以Test开头的函数都会被自动执行

# 标记单元测试结果
## FailNow
终止当前测试用例
{{< highlight go "linenos=inline" >}}
func TestSomeFunc(t *testing.T) {
    ...
    t.FailNow()
    ...
}
{{< /highlight >}}

## Fail
只标记错误，但不终止测试用例
{{< highlight go "linenos=inline" >}}
func TestSomeFunc(t *testing.T) {
    ...
    t.Fail()
    ...
}
{{< /highlight >}}

例如有以下两个测试用例:
{{< highlight go "linenos=inline" >}}
func TestSomeFuncFailNow(t *testing.T) {
    fmt.Println("before fail now")
    t.FailNow()
    fmt.Println("after fail now")
}

func TestSomeFuncFail(t *testing.T) {
    fmt.Println("before fail")
    t.Fail()
    fmt.Println("after fail")
}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
go test -v b_test.go
=== RUN   TestSomeFuncFailNow
before fail now
--- FAIL: TestSomeFuncFailNow (0.00s)
=== RUN   TestSomeFuncFail
before fail
after fail
--- FAIL: TestSomeFuncFail (0.00s)
FAIL
FAIL	command-line-arguments	0.006s
FAIL
{{< /highlight >}}

# 单元测试日志
每个测试用例可能并发执行，使用 testing.T 提供的日志输出可以保证日志跟随这个测试上下文一起打印输出。testing.T 提供了几种日志输出方法，testing.T 提供了几种日志输出方法，详见下表

|方法|使用说明|
|:----|:-----|
|Log|打印日志，go test -v时才显示|
|Logf|格式化打印日志，go test -v时才显示|
|Error|打印错误日志|
|Errorf|格式化打印错误日志|
|Fatal|打印致命日志，同时结束测试|
|Fatalf|格式化打印致命日志，同时结束测试|

示例：
{{< highlight go "linenos=inline" >}}
func TestSomeFuncLog1(t *testing.T) {
    t.Log("test t.Log")
    t.Error("test t.Error")
    t.Fatal("test t.Fatal")
}

go test b_test.go
--- FAIL: TestSomeFuncLog (0.00s)
    b_test.go:8: test t.Log
    b_test.go:9: test t.Error
    b_test.go:10: test t.Fatal
FAIL
FAIL	command-line-arguments	0.002s
FAIL
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
func TestSomeFuncLog2(t *testing.T) {
    t.Log("test t.Log")
    t.Fatal("test t.Fatal")
    t.Error("test t.Error")
}

go test b_test.go
--- FAIL: TestSomeFuncLog2 (0.00s)
    b_test.go:8: test t.Log
    b_test.go:9: test t.Fatal
FAIL
FAIL	command-line-arguments	0.001s
FAIL
{{< /highlight >}}

# 基准测试
基准测试可用于获取代码内存占用和运行效率相关的性能数据。基准测试用例必须以Benchmark开头，比如BenchmarkXxx。
## 测试CPU
{{< highlight go "linenos=inline" >}}
func Benchmark_Add(b *testing.B) {
    var n int
    for i := 0; i < b.N; i++ {
        n++
    }
}
{{< /highlight >}}
b.N由基准测试框架提供，用于构造压力测试的循环体，测试代码需要保证函数可重入性及无状态，确保不使用全局变量等带有记忆性质的数据结构

{{< highlight go "linenos=inline" >}}
go test -v -bench=. bench_test.go
goos: linux
goarch: amd64
Benchmark_Add 	1000000000	         0.382 ns/op
PASS
ok  	command-line-arguments	0.424s
{{< /highlight >}}
-bench=.表示运行bench_test.go文件中的所有基准测试，类似单元测试中的-run，Benchmark_Add为基准测试名称，1000000000表示测试次数，也就是testing.B
结构中提供给程序使用的N，0.382 ns/op表示每一个操作耗时0.382ns


## 测试内存
{{< highlight go "linenos=inline" >}}
func Benchmark_Alloc(b *testing.B) {
    for i := 0; i < b.N; i++ {
        fmt.Sprintf("%d", i)
    }
}
{{< /highlight >}}

通过在命令行中添加-benchmem参数就可以显示内存分配情况
{{< highlight go "linenos=inline" >}}
go test -v -bench=Alloc -benchmem bench_test.go
goos: linux
goarch: amd64
Benchmark_Alloc 	 9288526	       131 ns/op	      16 B/op	       2 allocs/op
PASS
ok  	command-line-arguments	2.317s
{{< /highlight >}}
16 B/op表示每一次调用需要分配16个字节，2 allocs/op表示每一次调用有两次分配，开发者根据这些信息可以迅速找到可能的分配点，进行优化和调整。


# 基准测试原理
基准测试框架对一个测试用例的默认测试时间是1秒。开始测试时，当以Benchmark开头的基准测试用例函数返回时还不到1秒，那么testing.B 中的 N 值将按 1、2、5、10、20、50……递增，同时以递增后的值重新调用基准测试用例函数

可以通过-benchtime参数自定义测试时间
{{< highlight go "linenos=inline" >}}
go test -v -bench=. -benchtime=30s bench_test.go
goos: linux
goarch: amd64
Benchmark_Add 	1000000000	         0.418 ns/op
PASS
ok  	command-line-arguments	0.465s
{{< /highlight >}}
貌似不管用，后面在看看

# 控制计时器
有些测试需要一定的启动和初始化时间，如果从 Benchmark() 函数开始计时会很大程度上影响测试结果的精准性。testing.B 提供了一系列的方法可以方便地控制计时器，从而让计时器只在需要的区间进行测试。我们通过下面的代码来了解计时器的控制。
{{< highlight go "linenos=inline" >}}
func Benchmark_Add_TimerControl(b *testing.B) {
    // 重置计时器
    b.ResetTimer()

    // 停止计时器
    b.StopTimer()

    // 开始计时器
    b.StartTimer()

    var n int
    for i := 0; i < b.N; i++ {
        n++
    }
}
{{< /highlight >}}
从Benchmark()函数开始，Timer就开始计数。StopTimer()可以停止这个计数过程，做一些耗时的操作，通过StartTimer() 重新开始计时。ResetTimer() 可以重置计数器的数据。
计数器内部不仅包含耗时数据，还包括内存分配的数据。

# 测试代码覆盖率
{{< highlight go "linenos=inline" >}}
go test -cover

PASS
coverage: 40.6% of statements
ok  	github.com/containerd/containerd/oci	0.010s
{{< /highlight >}}
需要特别说明的是，测试的覆盖度正常情况下是跑不满100%，比如说写的代码是来接住panic的等等异常的，那其实就不会走到了。

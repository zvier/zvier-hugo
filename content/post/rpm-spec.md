---
title: "rpm spec"
date: 2019-09-13T15:19:57+08:00
draft: true
categories: [""]
tags: ["linux"]
---
# 简述
如果软件项目中提供了spec文件，则可以将软件源码编译打包成rpm包，spec文件只是一个具有特殊语法的文本文件，文件中包含了很多软件包信息以及打包步骤。

<!--more-->
打包命令为:
{{< highlight bash "linenos=inline" >}}
rpmbuild -ba project.spec
{{< /highlight >}}
rpmbuild在根据spec文件打包时，会在主目录下创建一个rpmbuild目录作为打包时的工作目录，rpmbuild目录内的结构如下:
{{< highlight bash "linenos=inline" >}}
rpmbuild
├── BUILD
├── BUILDROOT
├── RPMS
├── SOURCES
├── SPECS
└── SRPMS
    └── test-1.1.1-00.src.rpm
{{< /highlight >}}
即包含5个子目录，每个目录分别存放打包时用到或输出的文件:
* BUILD: rpmbuild编译软件的目录
* BUILDROOT:
* RPMS: rpmbuild创建的binary RPM所存放的目录
* SOURCES: 存放源代码的目录
* SPECS: 存放spec文件的目录
* SRPMS: rpmbuild创建的source RPM所存放的目录


下面结合实例<code>k8s.io/kubernetes/build/rpms/kubelet.spec</code>，详解rpm spec文件的语法。  

# 注释
spec文件的注释以<code>#</code>开头，如果注释里有<code>%</code>，需要再附加一个<code>%</code>，比如: <code>%%prep</code>  

# spec文件头
spec文件头主要包含rpm包的名字、版本、类别、说明摘要等信息  
## Name(必须)
Name定义了rpm包的名称，后面可以使用<code>%{name}</code>引用，格式如下:
{{< highlight bash "linenos=inline" >}}
Name: <software name>
{{< /highlight >}}
例如kubelet.spec文件中的首行就定义了kubelet软件包包名
{{< highlight bash "linenos=inline" >}}
Name: kubelet
{{< /highlight >}}

## Version(必须)
Version定义了rpm包的版本号，后面可以使用<code>%{version}</code>引用，格式如下:
{{< highlight bash "linenos=inline" >}}
Version: <software vesion>
{{< /highlight >}}
例如kubelet.spec文件中的版本定义:
{{< highlight bash "linenos=inline" >}}
Version: OVERRIDE_THIS
{{< /highlight >}}

## Release(必须)
Release定义了rpm包的发行号，后面可以使用<code>%{release}</code>引用，格式如下:
{{< highlight bash "linenos=inline" >}}
Release: <software release>
{{< /highlight >}}
kubelet.spec文件为:
{{< highlight bash "linenos=inline" >}}
Release: 00
{{< /highlight >}}

## Packager
Packager域定义了rpm包的打包人，一般多人开发的项目中没有写  
{{< highlight bash "linenos=inline" >}}
Packager: zvier6@163.com
{{< /highlight >}}

## License(必须)
License域定义了rpm包的授权方式，可以是GPL(自由软件)，ASL 2.0等
{{< highlight bash "linenos=inline" >}}
License: <software license>
{{< /highlight >}}
kubelet.spec文件为:
{{< highlight bash "linenos=inline" >}}
License: ASL 2.0
{{< /highlight >}}

## Summary(必须)
Summary域定义了软件包的概要
{{< highlight bash "linenos=inline" >}}
Summary: <software summary>
{{< /highlight >}}

kubelet.spec文件为:
{{< highlight bash "linenos=inline" >}}
Summary: Container Cluster Manager - Kubernetes Node Agent
{{< /highlight >}}

## BuildRoot
BuildRoot定义了rpm包最终安装的目录，它是安装和编译时使用的一个虚拟根目录，在生成rpm包中，执行make
install时正是把软件安装到这个路径中，在打包时，同样依赖该虚拟根目录，后面可以用$RPM_BUILD_ROOT引用  
{{< highlight bash "linenos=inline" >}}
BuildRoot：%{_tmppath}/%{name}-%{version}-%{release}-buildroot
{{< /highlight >}}

## URL
URL域给出了软件的官方主页
{{< highlight bash "linenos=inline" >}}
URL: <software website>
{{< /highlight >}}

kubelet.spec文件为:
{{< highlight bash "linenos=inline" >}}
URL: https://kubernetes.io
{{< /highlight >}}

## Vendor
定义了rpm包所属发行商或打包组织的信息
{{< highlight bash "linenos=inline" >}}
Vendor: <RedFlag Co,Ltd>
{{< /highlight >}}

## Group
描述了软件包所属的类别，格式如下:   
{{< highlight bash "linenos=inline" >}}
Group:       <Applications/Multimedia>
{{< /highlight >}}
Group的取值可以类似通过如下命令获取:
{{< highlight bash "linenos=inline" >}}
cat  /usr/share/doc/rpm-4.11.3/GROUPS
{{< /highlight >}}
以下稍举几个例子:
### 娱乐/游戏
{{< highlight bash "linenos=inline" >}}
Group: Amusements/Games
{{< /highlight >}}
### 娱乐/图形
{{< highlight bash "linenos=inline" >}}
Group: Amusements/Graphics
{{< /highlight >}}
### 开发/语言
{{< highlight bash "linenos=inline" >}}
Group: Development/Languages
{{< /highlight >}}
### 开发/函数库
{{< highlight bash "linenos=inline" >}}
Group: Development/Libraries
{{< /highlight >}}

## SourceN
定义了源码包的名字，如果只有一个源码包，可以只写Source     
{{< highlight bash "linenos=inline" >}}
Source0:       %{name}-%{version}.tar.gz
{{< /highlight >}}


# 依赖关系
依赖关系定义了一个包正常工作所依赖的其他包，rpm包在升级、安装和删除时，要确保依赖关系得到满足，rpm包支持以下4种依赖:  

## Requires
定义当前包所依赖的其它包，语法如下:  
{{< highlight bash "linenos=inline" >}}
Requires: <software>
{{< /highlight >}}
一行中可以定义多个依赖
{{< highlight bash "linenos=inline" >}}
Requires: ebtables ethtool
{{< /highlight >}}
在指定依赖关系时，还可以指定版本号，支持的运算符有: <code>>=</code> <code><=</code> <code>></code> <code>></code> <code>=</code>，运算符两边需要用空格隔开，而不同的软件名也需要用空格隔开，如果没有指定版本号，则表示可以是任意版本    
{{< highlight bash "linenos=inline" >}}
Requires: iptables >= 1.4.21
{{< /highlight >}}

{{< highlight bash "linenos=inline" >}}
Requires: iptables >= 1.4.21 ebtables ethtool
{{< /highlight >}}

## PreReq
指定依赖包必须先安装
{{< highlight bash "linenos=inline" >}}
PreReq: capability>=version
{{< /highlight >}}

## Conflicts
指定冲突关系，例如下例指定了本包和大于等于2.0的bash包有冲突   
{{< highlight bash "linenos=inline" >}}
Conflicts: bash>=2.0
{{< /highlight >}}

## Provides
这个包能提供的功能  

## Obsoletes(过时的)
其他包提供的功能已经不推荐使用了，这通常是其他包的功能修改了，老版本不再推荐使用，可能在以后的版本中废弃   

## BuildRequires
编译时的包依赖  
{{< highlight bash "linenos=inline" >}}
BuildRequires: zlib-devel
{{< /highlight >}}

# 描述(%description)(必须)
定义rpm包的描述信息，可写在多个行上   
{{< highlight bash "linenos=inline" >}}
%description
The node agent of Kubernetes, the container cluster manager.
{{< /highlight >}}

# 预处理(%prep)
预处理阶段会将压缩包中归档的源代码解开，并打上补丁，为下一步的编译安装作准备。和后面的%build，%install阶段一样，除了可以执行RPM所定义的宏命令（以%开头）外，还可以执行SHELL命令。

## 解压(%setup)
%setup用于解压源码，并将当前目录切换源码解压之后产生的目录  
### 指定要切换的目录(-n)
%setup宏默认产生和切换到的目录是: %{name}-%{version}，支持用-n选项指定要切换的目录  
{{< highlight bash "linenos=inline" >}}
%setup -n<newdir>
{{< /highlight >}}

{{< highlight bash "linenos=inline" >}}
%setup -n%{name}-XXX
{{< /highlight >}}

### 关闭tar命令的详细输出(-q)
{{< highlight bash "linenos=inline" >}}
%setup -q
{{< /highlight >}}

在解源码包之前先创建目录(-c)
{{< highlight bash "linenos=inline" >}}
%setup -c -nnewdir
{{< /highlight >}}

包含多个源码包时，将第num个源码包解压(-b)
{{< highlight bash "linenos=inline" >}}
%setup -bnum
{{< /highlight >}}
解开第一个源码包程序文件
{{< highlight bash "linenos=inline" >}}
%setup -b0
{{< /highlight >}}

## 打补丁(%patch)
%patch用于将头部定义的补丁打到源码上，如果定义了多个补丁，可以用一个数字参数来指示应用哪个补丁。

### 打第N个补丁，等价于%patch -pN
{{< highlight bash "linenos=inline" >}}
%patchN
{{< /highlight >}}
分别指定使用第一个和第二个补丁文件
{{< highlight bash "linenos=inline" >}}
%patch -p0
%patch -p1
{{< /highlight >}}

### 关闭打补丁信息
{{< highlight bash "linenos=inline" >}}
%patch -p0 -s
{{< /highlight >}}

### 指定补丁扩展名(-b extension)
在加入补丁文件之前，将源文件名上加入extension扩展名。缺省源文件会加上.orig。
{{< highlight bash "linenos=inline" >}}

{{< /highlight >}}

### 删除补丁输出文件(-T)
-T 将所有打补丁时产生的输出文件删除

# 编译(%build)
定义编译软件包所要执行的命令，可以是宏也可以是shell，一般由多个make命令组成。

# 安装(%install)
安装阶段定义了打包安装软件时将要执行的命令，类似make install，用于将已编译好的软件安装到Buildroot定义的虚拟目录结构中，从而打包形成一个rpm包。

# 清理(%clean)
为了防止由于Buildroot中的旧文件而导致错误的打包，必须在安装新文件之前将 Buildroot 中任何现有的文件删除，在制作了在 Install 段落中安装的文件的打包之后，将运行 %clean，保证下次构建之前 Buildroot 被清空。 

%clean
{{< highlight bash "linenos=inline" >}}
rm -rf $RPM_BUILD_ROOT
{{< /highlight >}}

# 文件(%files)
定义软件包所包含的文件，分为三类：说明文档（doc），配置文件（config）及执行程序，还可定义文件存取权限，拥有者及组别。

这里也是在虚拟根目录下进行，不要写绝对路径，而应该用宏或变量表示相对路径。 如果描述为目录，表示目录中除%exclude外的所有文件。
%defattr (-,root,root) 指定包装文件的属性，分别是(mode,owner,group)，-表示默认值，对文本文件是0644，可执行文件是0755

 

# 更新日志(%changelog)
每次软件的更新内容可以记录在这里，保存到发布的软件包中，以便查询之用。

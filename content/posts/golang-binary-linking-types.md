+++
date = '2025-01-07T00:00:00+08:00'
draft = false
title = 'Golang 二进制文件的静态和动态链接'
+++


## 概述

Golang 最大的优势之一就是支持跨平台编译，真正的实现了Java的梦想：一次编译，到处运行。

不过，在某些特定情况下，也会存在一些特殊之处。

## 静态链接与动态链接

**静态链接**是是在编译时将程序与所依赖的所有库（通常是 `.a` 或 `.lib` 文件）一起打包，生成一个独立的可执行文件。也就是说，所有需要的函数和库代码在编译过程中就已经嵌入到可执行文件中。

因为这样编译出来的二进制更便携，它不需要在其运行的主机系统上存在库。因此，你的二进制文件可以在任何系统上运行，无论哪个发行版/版本，并且不依赖于任何系统库。

**动态链接**是在程序运行时（即执行时）将程序与共享库（如 `.dll`、`.so` 或 `.dylib` 文件）链接。程序本身包含对这些库的引用，而不直接包含库的代码。程序启动时，操作系统会加载这些库文件并链接到程序。

这种方式的优点是比较灵活，如果依赖的库发生了更新，只要接口没变，程序不需要编译就可以使用最新版本的库，而且因为二进制包含对库的引用，不包含库代码，因此文件较小。
## 静态链接程序

我们先来编写一个简单的 “Hello World” 程序，，经过打包后，它将成为一个静态链接的二进制文件。

```go
package main  
  
func main() {  
    println("hello world!")  
}
```

接着编译这个 Go 文件：

```bash
go build -o main1 main.go
```

然后使用 `file` 命令来检查文件类型：

```bash
$ file main1 | tr , '\n' 
hello_linux: ELF 64-bit LSB executable
 x86-64
 version 1 (SYSV)
 statically linked
 Go BuildID=...
 with debug_info
 not stripped
```

从上述命令的输出结果可知，`main1` 文件属于 **64 位静态链接的 ELF 格式** 可执行文件，适用于 **x86-64** 架构，包含了 **调试信息**，并且没有进行 **剥离**（即保留了调试符号）。

另外，还有 Linux 程序 `ldd` 能够帮助我们判断二进制文件是静态链接还是动态链接，执行以下命令：

```bash
$ ldd main1
not a dynamic executable
```

## 动态链接程序

在 Golang 中，有一种名为 CGO 的机制，它可以用来调用 C 代码，就连 Go 的标准库也在使用这一机制，例如 `net` 包（[https://pkg.go.dev/net](https://pkg.go.dev/net)），其就借助了 C 库来处理 DNS。以下是相关示例代码：

```go
package main  
  
import (  
    "fmt"  
    "log"    
    "net"
)  
  
func main() {  
    ns, err := net.LookupNS("typesafe.cn")  
    if err != nil {  
       log.Fatal(err)  
    }  
    for _, n := range ns {  
       fmt.Printf("%+v\n", n)  
    }  
}
```

同样需要编译这个 Go 文件：

```bash
go build -o main2 main2.go
```


之后，我们再次利用 `file` 和 `ldd` 程序来检查第二个二进制文件，操作及结果如下：

```bash
$ file main2 | tr , '\n'
main2: ELF 64-bit LSB executable
 x86-64
 version 1 (SYSV)
 dynamically linked
 interpreter /lib64/ld-linux-x86-64.so.2
 Go BuildID=...
 with debug_info
 not stripped
```

```bash
$ ldd main2
        linux-vdso.so.1 (0x00007ffc09ec8000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f3651936000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f3651b21000)
```

由此可以看出，`main2` 是一个 **64 位动态链接的 ELF 可执行文件**，其运行依赖于 **glibc**（`libc.so.6`）以及 **动态链接器**（`ld-linux-x86-64.so.2`），并且包含 **调试信息**，未进行剥离操作。在运行时，会通过 **动态链接器** 来加载这些共享库。

## 将动态链接转为静态链接

有意思的是，Go 的 `net` 包提供了纯 Go 实现，这使得我们在编译时可以禁用 `cgo`。具体而言，可以通过指定构建标签或者设置 **CGO_ENABLED=0** 来彻底禁用 `cgo`。示例如下：

```bash
$ CGO_ENABLED=0 go build -o main2 main2.go
$ ldd main2
not a dynamic executable
```

从上述输出结果能够证实，通过这两种方式操作后，最终都能得到一个静态二进制文件。
## 交叉编译

如文章开头所提到的，交叉编译是 Golang 的一大显著优势，它能够让代码在各种不同的平台与架构上运行。以下是一些常见的交叉编译命令示例：

```bash
# 编译为 macOS 上的 amd64 架构
GOOS=darwin GOARCH=amd64 go build -o main2_darwin_amd64 main2.go

# 编译为 macOS 上的 arm64 架构
GOOS=darwin GOARCH=arm64 go build -o main2_darwin_arm64 main2.go

# 编译为 Linux 上的 amd64 架构
GOOS=linux GOARCH=amd64 go build -o main2_linux_amd64 main2.go

# 编译为 Linux 上的 arm64 架构
GOOS=linux GOARCH=arm64 go build -o main2_linux_arm64 main2.go

# 编译为 Linux 上的 386 架构 (32-bit)
GOOS=linux GOARCH=386 go build -o main2_linux_386 main2.g
...
```

需要注意的是，前提是没有使用 CGO，一旦使用了 CGO，那么运行平台基本就会和开发平台绑定起来了。

## 减少二进制大小

或许你已经留意到，在上述 `file` 命令输出结果中出现了 “with debug_info not stripped” 这样的提示，这意味着二进制文件包含了调试信息。通常情况下，我们并不需要这些调试信息，将其删除能够有效减小二进制文件的大小，示例操作及结果如下：

```bash
$ CGO_ENABLED=0 go build -o main2 main.go
$ ll -h main2 
-rwxr-xr-x 1 root root 2.9M  1月 5日 15:32 main2
$ file main2 | tr , '\n'
main2: ELF 64-bit LSB executable
 x86-64
 version 1 (SYSV)
 statically linked
 Go BuildID=...
 with debug_info
 not stripped
$ CGO_ENABLED=0 go build -ldflags "-s -w" -o main2 main.go
$ ll -h main2 
-rwxr-xr-x 1 root root 2.0M  1月 5日 15:33 main2
$ file main2 | tr , '\n'
main2: ELF 64-bit LSB executable
 x86-64
 version 1 (SYSV)
 statically linked
 Go BuildID=...
 stripped
```

## 结语

静态链接和动态链接各有千秋，根据自身的业务选择合适的方案即可。
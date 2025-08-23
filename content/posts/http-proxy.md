+++
date = '2025-01-02T00:00:00+08:00'
draft = false
title = 'HTTP代理的原理和实现'
+++

HTTP 代理可以说是每个开发者都绕不开的工具。几乎每天都会使用，但你真的了解 HTTP 代理的原理吗？

> **说明**：这里讨论的 HTTP 代理是指 HTTP Proxy Server，具体是正向 HTTP 代理服务端的原理和实现。

想了解 HTTP 代理的原理，最严谨的方法是阅读 RFC 文档，但这同时也是最困难的方式。今天，我将介绍一种更直观的学习技巧。

从名字上就可以看出，HTTP 代理基于 HTTP 协议，而 HTTP 协议运行在 TCP 之上。我们可以直接监听某个 TCP 端口，模拟 HTTP 代理服务端，同时使用 `curl` 命令通过代理访问网站，观察具体数据，再决定如何实现这个 HTTP 代理服务端。

---

## 代理 HTTP 请求

**步骤 1**：使用 `ncat` 监听端口 7788。

```bash
ncat -lvp 7788
```

**步骤 2**：执行 `curl` 命令，通过代理访问百度的 HTTP 协议。

```bash
curl http://baidu.com/123?a=1 --proxy 127.0.0.1:7788
```

**步骤 3**：查看 `ncat` 的输出：

```
Ncat: Connection from 127.0.0.1:58458.
GET http://baidu.com/123?a=1 HTTP/1.1
Host: baidu.com
User-Agent: curl/8.7.1
Accept: */*
Proxy-Connection: Keep-Alive
```

第一行是 `ncat` 输出的信息，可以忽略。后面的内容完全是 HTTP 协议的数据。

由此可见，HTTP 代理实际上是一个“传话筒”：客户端将完整的请求协议发送给 HTTP 代理，代理转发给目标服务器，并将响应返回给客户端。

接下来，我们用 Golang 实现这种代理功能。

---

## 使用 Golang 实现 HTTP 代理

### 1. 搭建 HTTP 服务

以下代码使用 Golang 官方库启动 HTTP 服务，并通过一个 `handler` 处理所有请求：

```go
package main

import (
	"io"
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		handleHttp(w, r)
	})

	err := http.ListenAndServe(":7788", handler)
	if err != nil {
		log.Fatal(err)
	}
}
```

### 2. 实现代理逻辑

接下来实现代理的核心功能：

```go
func handleHttp(w http.ResponseWriter, r *http.Request) {
	// 向目标服务器发送请求
	resp, err := http.DefaultTransport.RoundTrip(r)
	if err != nil {
		http.Error(w, err.Error(), http.StatusServiceUnavailable)
		return
	}
	defer resp.Body.Close()

	// 将响应头和响应体写入客户端
	for k, vv := range resp.Header {
		for _, v := range vv {
			w.Header().Add(k, v)
		}
	}
	w.WriteHeader(resp.StatusCode)
	_, _ = io.Copy(w, resp.Body)
}

```

### 测试结果

运行代码后，再次执行 `curl` 命令：

```bash
curl http://baidu.com/123?a=1 --proxy 127.0.0.1:7788
```

**输出：**

```html
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>302 Found</title>
</head><body>
<h1>Found</h1>
<p>The document has moved <a href="http://www.baidu.com/search/error.html">here</a>.</p>
</body></html>

```

可以看到，HTTP 代理已经正确转发了请求。但目前只支持 HTTP 请求，接下来我们继续支持 HTTPS 协议。

---

## 代理 HTTPS 请求

**步骤 1**：使用 `ncat` 监听端口 7788。

```bash
ncat -lvp 7788
```

**步骤 2**：执行 `curl` 命令，通过代理访问百度的 HTTPS 协议。

```bash
curl https://baidu.com/123?a=1 --proxy 127.0.0.1:7788
```

**步骤 3**：查看 `ncat` 的输出：

```
CONNECT baidu.com:443 HTTP/1.1
Host: baidu.com:443
User-Agent: curl/8.7.1
Proxy-Connection: Keep-Alive
```

在之前的 HTTP 请求中，我们能够获取到请求方法、路径、参数等完整信息。然而，在当前的 HTTPS 请求中，仅能看到 `CONNECT` 方法以及目标服务器的主机名和端口，其余信息完全无法获取。

为什么会出现这样的情况？我们需要参考 RFC 定义。

在 RFC 9110 章节 9.3.6 - CONNECT 方法： https://www.rfc-editor.org/rfc/rfc9110.html#name-connect

> The CONNECT method requests that the recipient establish a tunnel to the destination origin server identified by the request target and, if successful, thereafter restrict its behavior to blind forwarding of data, in both directions, until the tunnel is closed. Tunnels are commonly used to create an end-to-end virtual connection, through one or more proxies, which can then be secured using TLS (Transport Layer Security[TLS13]).

翻译如下：

> CONNECT 方法请求接收者建立到由请求目标标识的目的地源服务器的隧道，如果成功，则将其行为限制为双向盲目转发数据，直到隧道关闭。隧道通常用于通过一个或多个代理创建端到端虚拟连接，然后可以使用 TLS（传输层安全性）保护该连接 [TLS13]。
### 使用 Golang 实现隧道功能

```go
func handleTunnel(w http.ResponseWriter, r *http.Request) {
	remoteConn, err := net.DialTimeout("tcp", r.Host, 5*time.Second)
	if err != nil {
		http.Error(w, err.Error(), http.StatusServiceUnavailable)
		return
	}
	defer remoteConn.Close()

	// 响应客户端隧道建立成功
	w.WriteHeader(http.StatusOK)

	// 获取客户端连接
	hijacker, ok := w.(http.Hijacker)
	if !ok {
		http.Error(w, "hijack not supported", http.StatusInternalServerError)
		return
	}
	clientConn, _, err := hijacker.Hijack()
	if err != nil {
		http.Error(w, err.Error(), http.StatusServiceUnavailable)
		return
	}
	defer clientConn.Close()

	// 双向拷贝数据
	go bidirectionalCopy(remoteConn, clientConn)
}
```

辅助函数：

```go
func bidirectionalCopy(conn1, conn2 net.Conn) {
	done := make(chan struct{})
	go func() {
		_, _ = io.Copy(conn1, conn2)
		done <- struct{}{}
	}()
	go func() {
		_, _ = io.Copy(conn2, conn1)
		done <- struct{}{}
	}()
	<-done
	<-done
}
```

---

## 完整代码

最终实现如下：
```go
package main  
  
import (  
    "io"  
    "log"    
    "net"    
    "net/http"    
    "time"
)  
  
func main() {  
    var handler = http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {  
       if r.Method == http.MethodConnect {  
          handleTunnel(w, r)  
       } else {  
          handleHttp(w, r)  
       }  
    })  
  
    err := http.ListenAndServe(":7788", handler)  
    if err != nil {  
       log.Fatal(err)  
    }  
}  
  
func handleHttp(w http.ResponseWriter, r *http.Request) {  
    // 向下游服务器发送请求  
    resp, err := http.DefaultTransport.RoundTrip(r)  
    if err != nil {  
       http.Error(w, err.Error(), http.StatusServiceUnavailable)  
       return  
    }  
    defer resp.Body.Close()  
  
    // 将响应头和响应体写入到客户端  
    for k, vv := range resp.Header {  
       for _, v := range vv {  
          w.Header().Add(k, v)  
       }  
    }  
    w.WriteHeader(resp.StatusCode)  
    _, _ = io.Copy(w, resp.Body)  
}  
  
func handleTunnel(w http.ResponseWriter, r *http.Request) {  
    remoteConn, err := net.DialTimeout("tcp", r.Host, 5*time.Second)  
    if err != nil {  
       http.Error(w, err.Error(), http.StatusServiceUnavailable)  
       return  
    }  
    defer remoteConn.Close()  
    // 响应客户端隧道建立成功  
    w.WriteHeader(http.StatusOK)  
  
    // 获取客户端长连接  
    hijacker, ok := w.(http.Hijacker)  
    if !ok {  
       http.Error(w, "hijack not supported", http.StatusInternalServerError)  
       return  
    }  
    centralConn, _, err := hijacker.Hijack()  
    if err != nil {  
       http.Error(w, err.Error(), http.StatusServiceUnavailable)  
    }  
  
    // 双向拷贝数据  
    go bidirectionalCopy(remoteConn, centralConn)  
}  
  
func bidirectionalCopy(conn1, conn2 net.Conn) {  
    done := make(chan struct{})  
    go func() {  
       _, _ = io.Copy(conn1, conn2)  
       done <- struct{}{}  
    }()  
  
    go func() {  
       _, _ = io.Copy(conn2, conn1)  
       done <- struct{}{}  
    }()  
  
    <-done  
    <-done  
  
    _ = conn1.Close()  
    _ = conn2.Close()  
}
```
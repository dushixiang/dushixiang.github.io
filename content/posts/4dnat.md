---
title: "服务器不允许上网并且需要跳板机才能访问？学会使用这个工具，轻松让服务器使用yum。"
categories: [ "tool" ]
tags: ["4dnat","linux","windows","dnat"]
draft: false
slug: "4dnat"
date: "2020-11-19 22:33:00"
---

# 前言

你是否遇到过这样的场景，服务器不能上网，但是又需要安装某个软件，面对如蛛网般杂乱的rpm包依赖关系，放弃或许是最好的选择，这样你就不必再为无法完成工作而痛苦又懊恼。

但是今天，你有了一个更好的选择。

# [4DNAT](https://github.com/dushixiang/4dnat)

4DNAT取名源自4和DNAT。这个工具工作在OSI模型的第四层传输层，同时4和for谐音，意为专门为目标地址转换而服务的工具。4DNAT使用go语言开发，具有天然的跨平台性，并且完全使用go标准库开发，没有任何的第三方依赖，编译之后只有一个二进制可执行文件。它有4种工作模式：

### 转发模式

接受两个参数，监听端口和目标地址，在监听端口接收到请求后会主动连接目标地址，示例：

```bash
./4dnat -forward 2222 192.168.1.100:22
```

### 监听模式

接受两个参数，监听端口1和监听端口2，并交换两个端口接收到的数据，示例：

```bash
./4dnat -listen 10000 10001
```

### 代理人模式

接受两个参数，目标地址1和目标地址2，启动后会主动连接这两个目标地址，并交换两个端口接收到的数据，示例：

```bash
./4dnat -agent 127.0.0.1:10000 127.0.0.1:22
```

### http/https代理模式

接受两个参数或四个参数，代理类型、监听端口、证书路径和私钥路径，示例：

#### http代理

```shell script
./4dnat -proxy http 1080
```

#### https代理

```shell script
./4dnat -proxy https 1080 server.crt server.key
```

# 使用场景

### 场景一

期望可以在**用户电脑**上直接访问目标服务器上的3306端口，跳板机器是一台Windows机器，没办法做ssh端口转发。
![请输入图片描述](https://oss.typesafe.cn/break-through-the-network.png)

> 单向虚线箭头表示可以单向访问，反之不行。

使用4DNAT在**跳板机器**上执行如下命令做端口转发

  ```bash
  # 本地监听3307端口，接收到请求后主动连接10.1.0.40的3306端口
  ./4dnat -forward 3307 10.1.0.40:3306
  ```

在**用户电脑**上访问**172.16.0.30:3307**即等同于访问**10.1.0.40:3306**，于是就可以在**用户电脑**愉快的访问**目标机器**上的服务啦。

### 场景二

期望目标**目标机器**可以上网，如使用yum安装软件。
  ![请输入图片描述](https://oss.typesafe.cn/break-through-the-network2.png)

1. 在**用户电脑**上开启一个http代理
  ```bash
  ./4dnat -proxy http 1080
  ```

2. 在**跳板机器**上使用监听模式监听两个端口，用于交换数据
  ```bash
  ./4dnat -listen 10000 10001
  ```
3. 在**目标机器**上使用监听模式监听两个端口，用于交换数据
  ```bash
  ./4dnat -listen 20000 20001
  ```
4. 在**用户电脑**上使用代理人模式主动连接两个目标地址，用于交换数据
  ```bash
  ./4dnat -agent 127.0.0.1:1080 172.16.0.30:10000
  ```

5. 在**跳板机器**上使用代理人模式主动连接两个目标地址，用于交换数据
  ```bash
  ./4dnat -agent 127.0.0.1:10001 10.1.0.40:20000
  ```
6. 在**目标机器**上修改代理
  ```bash
  cat <<EOF >> /etc/profile
  http_proxy=http://127.0.0.1:20001
  https_proxy=http://127.0.0.1:20001
  export http_proxy https_proxy
  EOF

  source /etc/profile
  ```
7. 在**目标机器**上测试访问互联网
  ```bash
  curl https://typesafe.cn
  ```

最后奉上项目地址 https://github.com/dushixiang/4dnat
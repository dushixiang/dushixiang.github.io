---
title: "深入理解 Linux 虚拟网络设备 tun/tap"
description: "本文深入浅出地讲解了 Linux 中的虚拟网络设备 tun 和 tap，内容涵盖其基本概念、工作原理、两者差异、以及实际操作示例，帮助你更好地理解和使用它们。"
categories: [ "Linux网络" ]
tags: [ "linux","network","tun/tap", "虚拟网络" ]
draft: false
slug: "linux-tun-tap"
date: "2020-11-08 22:11:00"
---

在探索Linux网络虚拟化的世界时，你一定会遇到tun和tap这两个关键概念。它们是理解VPN、虚拟机网络、容器网络等技术的基础。本文将带你深入了解tun/tap是什么，它们是如何工作的，以及如何在实践中使用它们。

> 在计算机网络中，**tun**与**tap**是操作系统内核中的虚拟网络设备。不同于普通靠硬件网络适配器实现的设备，这些虚拟的网络设备全部用软件实现，并向运行于操作系统上的软件提供与硬件的网络设备完全相同的功能。

### tun/tap是什么？

- **tun** (tunnel的缩写) 是一个虚拟的点对点网络设备，工作在网络层（OSI模型的第三层）。它处理的是IP数据包，所以你可以把它看作是一个虚拟的网卡，但它没有物理的MAC地址。tun设备常用于实现各种IP隧道，例如OpenVPN和IPSec。

- **tap** (network tap的缩写) 是一个虚拟的以太网设备，工作在数据链路层（OSI模型的第二层）。它处理的是以太网帧，因此它拥有一个MAC地址，行为上更像一个真实的以太网卡。tap设备最常见的用途是作为虚拟机的网卡（如QEMU/KVM），或者用于创建网络桥接，将虚拟机接入物理局域网。

### tun/tap 的工作原理

Linux中的tun/tap设备提供了一种能力，让用户空间的应用程序能够像读写文件一样，直接向内核网络协议栈注入数据包，或者从协议栈中接收数据包。

操作tun/tap设备主要通过一个特殊的字符设备文件 `/dev/net/tun`。当一个应用程序打开这个文件时，内核会创建一个与该文件描述符关联的虚拟网络接口（如 `tun0` 或 `tap0`）。

数据流动的过程可以用下图来概括：

```mermaid
graph TD
    subgraph "用户空间 (User Space)"
        App("应用程序 (e.g., VPN, QEMU)")
    end

    subgraph "内核空间 (Kernel Space)"
        DevFile("/dev/net/tun")
        Driver("TUN/TAP 驱动")
        NetStack("网络协议栈")
        PhyDriver("物理网卡驱动")
    end
    
    PHY("物理网卡")

    App <-->|经由文件描述符<br/>read()/write()| DevFile
    DevFile <--> Driver
    Driver <--> NetStack
    NetStack <--> PhyDriver
    PhyDriver <--> PHY
```

**数据发送流程 (Outbound):**

1.  用户空间的应用程序（例如VPN客户端）通过文件描述符向 `/dev/net/tun` 写入一个IP包（对于tun设备）或以太网帧（对于tap设备）。
2.  tun/tap驱动接收到数据，并将其作为一个数据包注入到内核网络协议栈中，就像数据是从一个真实的物理网卡传来一样。
3.  网络协议栈根据其路由表等信息对数据包进行处理（例如，路由选择、NAT转换）。
4.  最终，协议栈将数据包通过真实的物理网卡发送出去。

**数据接收流程 (Inbound):**

1.  物理网卡接收到一个数据包。
2.  内核网络协议栈处理这个数据包。
3.  如果路由规则指示这个数据包应该被发送到tun/tap虚拟接口，协议栈就会将它交给tun/tap驱动。
4.  驱动程序将该数据包放入一个队列，等待用户空间的应用程序通过文件描述符从 `/dev/net/tun` 中读取。

> **注意**：当应用程序关闭该文件描述符时，对应的虚拟网络接口以及相关的路由等信息也会被内核自动删除。

### tun 与 tap 的核心区别

虽然tun和tap的工作原理相似，但它们工作的网络层次不同，这导致了它们用途上的巨大差异。

| 特性             | TUN 设备                                      | TAP 设备                                                 |
| ---------------- | --------------------------------------------- | -------------------------------------------------------- |
| **工作层级**     | 网络层 (L3)                                   | 数据链路层 (L2)                                          |
| **处理数据单元** | IP 包 (IP Packet)                             | 以太网帧 (Ethernet Frame)                                |
| **是否有MAC地址**| 否                                            | 是                                                       |
| **通信方式**     | 点对点 (Point-to-Point)                       | 类似以太网广播 (Ethernet-like)                           |
| **常见用途**     | 路由模式的VPN (如 OpenVPN), IP隧道            | 桥接模式的VPN, 虚拟机网卡 (QEMU/KVM), 容器网络         |

简而言之，如果你只需要在IP层面对流量进行隧道传输，并且不需要关心二层的广播和MAC地址，tun是更高效的选择。如果你需要一个完整的虚拟以太网卡，例如让虚拟机像物理机一样接入局域网，那么tap是必需的。

### 如何创建和使用 tun/tap 设备

Linux提供了`ip tuntap`命令来方便地创建持久化的tun/tap设备。

`ip tuntap`的基本用法如下:
```bash
# 查看帮助
ip tuntap help

# Usage: ip tuntap { add | del | show | list | lst | help } [ dev PHYS_DEV ]
# 	[ mode { tun | tap } ] [ user USER ] [ group GROUP ]
# 	[ one_queue ] [ pi ] [ vnet_hdr ] [ multi_queue ] [ name NAME ]
```

#### 示例：创建和配置 `tun` 设备

tun设备通常用于点对点连接，比如VPN。

```bash
# 1. 创建一个名为 tun0 的 tun 设备
# 需要 root 权限
sudo ip tuntap add dev tun0 mode tun

# 2. 启用 tun0 设备
sudo ip link set dev tun0 up

# 3. 为 tun0 分配一个 IP 地址
# 这是一个点对点设备，我们可以这样为其配置IP
sudo ip addr add 10.0.0.1 peer 10.0.0.2 dev tun0

# 4. 查看设备信息
ip addr show tun0
# 输出示例:
# tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
#     link/none
#     inet 10.0.0.1 peer 10.0.0.2/32 scope global tun0
#        valid_lft forever preferred_lft forever

# 5. 测试
# 此时，内核已经知道如何将发往 10.0.0.2 的包路由到 tun0 设备。
# 如果有一个程序正在监听 tun0，它就会收到这些 ping 包。
ping -c 3 10.0.0.2

# 6. 删除设备
sudo ip tuntap del dev tun0 mode tun
```

#### 示例：创建和配置 `tap` 设备

tap设备如同一个虚拟以太网卡，常用于网桥。

```bash
# 1. 创建一个名为 tap0 的 tap 设备
sudo ip tuntap add dev tap0 mode tap

# 2. 启用 tap0 设备
sudo ip link set dev tap0 up

# 3. 为 tap0 分配一个 IP 地址 (可选，通常由桥接管理)
sudo ip addr add 192.168.10.1/24 dev tap0

# 4. 查看设备信息，可以看到它有一个MAC地址
ip addr show tap0
# 输出示例:
# tap0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
#     link/ether 1a:2b:3c:4d:5e:6f brd ff:ff:ff:ff:ff:ff
#     inet 192.168.10.1/24 scope global tap0
#        valid_lft forever preferred_lft forever

# tap 设备最强大的地方在于可以桥接。
# 例如，可以创建一个网桥 br0，将 tap0 和物理网卡 eth0 连接起来，
# 这样连接到 tap0 的虚拟机就能和局域网中的其他机器无缝通信。

# 5. 删除设备
# 也可以使用 ip link 命令删除
sudo ip link del tap0
```

### 总结

tun和tap是Linux网络虚拟化中强大而基础的工具。理解它们在OSI模型中所处的层次是掌握其用法的关键：tun是网络层的管道，用于传输IP包；tap是链路层的虚拟网卡，用于传输以太网帧。希望通过本文的讲解，你对它们有了更清晰的认识。
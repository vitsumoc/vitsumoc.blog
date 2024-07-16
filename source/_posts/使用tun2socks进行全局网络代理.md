---
title: 使用tun2socks进行全局网络代理
url: 使用tun2socks进行全局网络代理
date: 2024-04-29 10:20:55
categories:
- 网络工具
tags:
- 网络工具
---

`tun2socks` (tunnel to socks) 是一种将 tunnel 流量转发到 socks 代理的工具，通常可以用来进行全局流量代理，也可以配合路由表设置，进行一些特定网段的流量代理。

<!-- more -->

# 简介

在介绍 `tun2socks` 之前，需要先了解一些基本概念的含义：

- tunnel：来自于 TUN/TAP，其中的 TUN 表示 tunnel，可以理解为三层代理，而 TAP 可以被理解为二层代理。
- socks：一种应用层代理技术，将进入其中的流量代理到远端服务器。
- 虚拟网卡：由 TUN/TAP 实现的，操作系统创建的虚拟网络设备，可以像普通网卡一样处理应用程序的流量。
- tunnel2socks：连通虚拟网卡和 socks 服务的工具。

以下是 [xjasonlyu/tun2socks](https://github.com/xjasonlyu/tun2socks) 项目中的功能介绍，后续的示例中我们也会使用该项目作为例子：

- 全局代理: 处理来自本设备的任意网络应用的所有网络流量并通过代理转发。
- 代理协议: 通过 HTTP/Socks4/Socks5/Shadowsocks 远程连接且支持鉴权。
- 跨平台性: 具有 Linux/macOS/Windows/FreeBSD/OpenBSD 特定优化的多平台支持。
- 网关模式: 作为第三层网关处理来自同一网络中其他设备的所有网络流量。
- IPv6 支持: 所有功能都可以在 IPv6 中工作，允许通过 IPv6 代理转发 IPv4 连接，反之亦然。
- TCP/IP 栈: 由来自 Google 容器应用程序内核 [gVisor](https://github.com/google/gvisor) 的用户空间 TCP/IP 网络栈强力驱动。

# 环境

为了方便理解，介绍一下笔者所在的网络环境，和本次实验相关的设备共有 3 台：

- 笔记本：
  - windows系统
  - 地址 `192.138.34.17/24`
  - 网关 `192.168.34.1`，网关是一台普通路由器
  - 有互联网连接
- 内网服务器：
  - linux系统
  - 地址 `192.168.34.197/24`
  - 网关为 `192.168.34.1`
  - 有互联网连接
  - 运行着一个 socks5 服务，服务地址为 `0.0.0.0:1080`，socks5 服务和远端服务器已连接
- 外网服务器：
  - linux系统
  - 地址为 `xx.xx.xx.xx`，负责处理通过内网服务器 socks5 服务发来的流量

# 测试

接下来的测试按照 xjasonlyu/tun2socks 项目提供的 [示例](https://github.com/xjasonlyu/tun2socks/wiki/Examples) 进行。

#### 下载 wintun

[wintun](https://www.wintun.net/) 是一个在 windows 系统中创建三层 TUN 的库，只需下载 dll 文件并放在 `tun2socks` 同文件夹下。

#### 创建虚拟网卡并设置代理

> tun2socks -device wintun -proxy socks5://192.168.34.197:1080 -interface "以太网"

以上命令会开启 `tun2socks` 进程，创建名为 `wintun` 的虚拟网卡，并将通过此网卡的流量通过指定的 `socks5` 服务器代理，通过原有 `以太网` 网卡转发。

#### 配置虚拟网卡

在新的命令行窗口中键入命令，进行虚拟网卡配置：

> netsh interface ipv4 set address name="wintun" source=static addr=192.168.123.1 mask=255.255.255.0

> netsh interface ipv4 set dnsservers name="wintun" static address=8.8.8.8 register=none validate=no

上述两条命令设置网卡的地址、掩码、DNS 服务器，让他看起来像一个正常的网卡。

#### 配置默认路由

> netsh interface ipv4 add route 0.0.0.0/0 "wintun" 192.168.123.1 metric=1

通过上述命令添加一条经过虚拟网卡的默认路由，之后全局流量都会通过虚拟网卡转发。

# 问题

- 测试时使用了一个不支持 UDP 的 socks5 代理，导致 DNS 解析功能受阻，如果想使用完整全局代理功能，需要另选 socks5 代理软件。

后来使用支持 udp_over_tcp 的代理套了一层，解决了惯用 socks 代理不能代理 UDP 的问题。

# 参考

- [tun2socks](https://github.com/xjasonlyu/tun2socks)
- [wintun](https://www.wintun.net/)
- [[教程] 在 Windows 上使用 tun2socks 进行全局代理](https://tachyondevel.medium.com/%E6%95%99%E7%A8%8B-%E5%9C%A8-windows-%E4%B8%8A%E4%BD%BF%E7%94%A8-tun2socks-%E8%BF%9B%E8%A1%8C%E5%85%A8%E5%B1%80%E4%BB%A3%E7%90%86-aa51869dd0d)
- [TUN/TAP](https://en.wikipedia.org/wiki/TUN/TAP)
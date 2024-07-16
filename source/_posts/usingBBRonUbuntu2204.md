---
title: "在 Ubuntu 22.04 LTS 上使用 BBR"
url: usingBBRonUbuntu2204
date: 2024-04-11 12:27:40
categories:
- 网络工具
tags:
- 网络工具
- 豆知识
- Linux
- VPS
---

# 什么是BBR

> 以下内容摘自[Wiki](https://en.wikipedia.org/wiki/TCP_congestion_control#TCP_BBR)

<!-- more -->

Bottleneck Bandwidth and Round-trip propagation time (BBR) 是由 Google 在 2016年开发的一种拥塞控制算法 (CCA)。虽然其他的拥塞控制算法大多基于丢包机制，通过丢包来检测到拥塞并降低传输速率，但 BBR，和 TCP Vegas 相似，是基于模型的。算法利用网络传送最近一次发出的数据包达到的最大带宽和往返时间来构建网络模型。每次累积确认或选择性确认数据包的传送都会产生一个速率样本，该样本记录了数据包的发送时间和确认时间之间间隔内传送的数据量。

在 YouTube 的实际部署中，BBRv1 版本平均带来了 4% 的网络吞吐量提升，在某些国家甚至能达到 14%。BBR 算法从 Linux 4.9 版本开始就被集成到 Linux 内核的 TCP 协议栈中。它同样也适用于 QUIC 协议。

BBR 版本 1 (BBRv1) 对非 BBR 流的公平性存在争议。尽管谷歌的演示文稿显示 BBRv1 可以很好地与 CUBIC 协同工作，但一些研究人员 (例如 Geoff Huston 和 Hock、Bless 及 Zitterbart) 认为它对其他数据流并不公平，并且难以扩展。Hock等人还发现 Linux 4.9 版本的 BBR 实现存在一些严重的固有缺陷，例如增加的队列延迟、不公平性和大量丢包。Soheil Abbasloo 等人 (C2TCP 算法的作者) 指出，BBRv1 版本在动态环境（例如蜂窝网络）中表现不佳。他们还指出 BBR 存在公平性问题。例如，当网络中存在一个 CUBIC 流 (它是 Linux、Android 和 MacOS 操作系统默认的 TCP 实现) 和一个 BBR 流时，BBR 流可能会主导 CUBIC 流，并占用整个链路带宽。

第 2 版 (Version 2) 尝试解决与基于丢包的拥塞控制 (例如 CUBIC) 共同运行时出现的不公平问题。BBRv2 改进了 BBRv1 的模型，加入了丢包信息和来自显式拥塞通知 (ECN) 的信息。虽然 BBRv2 有时可能比 BBRv1 吞吐量更低，但它通常被认为具有更好的有效吞吐量 (goodput)。\[需要来源\]

第 3 版 (BBRv3) 修复了 BBRv2 中的两个漏洞（带宽探测过早结束、带宽收敛问题）并进行了一些性能调整。它还提供了一个变体版本，称为 BBR.Swift，专为数据中心内部链路进行优化：它使用 network_RTT（不包含接收方延迟）作为主要的拥塞控制信号。

# 使用

既然 BBR 已经被集成到 Linux 内核中，那么我们稍作配置就可以使用，首先：

```bash
echo net.core.default_qdisc=fq >> /etc/sysctl.conf
echo net.ipv4.tcp_congestion_control=bbr >> /etc/sysctl.conf
```

向 `sysctl.conf` 写入两条配置：

- `net.core.default_qdisc=fq` 修改 Linux 系统中的的 qdisc 为 fq(fair queue)，避免 BBR 占用太多流量
- `net.ipv4.tcp_congestion_control=bbr` >> 将 ipv4 的 TCP 拥塞控制算法设置为 BBR

接下来，保存生效：

```bash
sysctl -p
```

查看是否已经应用：

```bash
sysctl net.ipv4.tcp_congestion_control
net.ipv4.tcp_congestion_control = bbr
```

这样就完成了所有配置。

（经过实际测试，不开启公平队列时 BBR 能提供更大的速率，也许对于一些纯粹跑流量的服务器（例如网络代理），不开启公平队列是更好的用法。）
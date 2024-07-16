---
title: 使用n2n连接不同局域网设备
url: n2nBeginner
date: 2023-12-06 14:37:00
categories:
- 网络工具
tags:
- 网络工具
- 网络编程
---

# 前言

目的是想在办公室使用家里的服务器

家里有不固定的公网IP，办公室有固定的公网IP，因此打算使用办公室服务器做Server

为什么不用frp？：因为想获得一个完整的网络服务，而frp只能做端口映射，如果开发过程中新增端口，需要修改frp就很麻烦

<!-- more -->

# n2n简介

`n2n` 是一个开源项目，地址在这里：

https://github.com/ntop/n2n

`n2n` 是一个二层VPN技术，他能在家里的服务器和办公室的服务器之间创建一个局域网链接

`n2n` 网络由 ```supernode``` 和 ```edge``` 组成，可以简单理解为同一 ```supernode``` 下的所有 ```edge``` 都处在同一个局域网中。

# 网络环境

## 办公室网络：

+ 网段：192.168.34.0/24
+ 网关：192.168.34.1
+ 网关公网地址：88.88.88.88
+ 服务器地址：192.168.34.194

## 家庭网络

+ 网段：192.168.0.0/24
+ 网关：192.168.0.1
+ 网关公网地址：不固定
+ 服务器地址：192.168.0.12

## 规划n2n网络

由于办公室有固定的公网地址，就由办公室服务器充当 `supernode`，同时家庭服务器和办公室服务器都是此 `supernode` 下的 `edge`
`n2n` 会形成一个新的局域网，规划如下：

+ 网段：10.0.34.0/24
+ 网关：无
+ 办公室服务器：10.0.34.21
+ 家庭服务器：10.0.34.41
  
# 实施

## 下载安装n2n

在办公室服务器和家庭服务器都下载并安装 `n2n`：

https://github.com/ntop/n2n/releases

安装完成后，服务器中会自动生成两个服务 `supernode` 和 `edge`

配置文件位于 `/etc/n2n/`

![](installVerify.png)

## 配置办公室服务器

办公室服务器需要承担三个职能：充当 `supernode`，充当 `edge`，转发其他办公室设备到家庭服务器的网络包

### 配置supernode

配置 `/etc/n2n/community.list` 文件，指定community名称

```text community.list
com8888 # community名称
```

复制 `supernode.conf.sample` 文件，并修改配置内容

```bash bash
cp /etc/n2n/supernode.conf.sample /etc/n2n/supernode.conf
```

```bash bash
vi /etc/n2n/supernode.conf  
```

```conf supernode.conf
-p=7777 # 指定supernode服务端口
-c=/etc/n2n/community.list # 指定引用的community文件
```

启动supernode

```shell shell
systemctl enable supernode
systemctl start supernode
```

之后可以看到 `supernode` 已经启动，并且在7777端口提供服务：

![](supernodeVerify.png)

### 配置edge

配置 `edge` 使办公室服务器成为 `n2n` 网络的成员

复制 `edge.conf.sample` 文件，并修改配置内容

```bash bash
cp /etc/n2n/edge.conf.sample /etc/n2n/edge.conf
```

```bash bash
vi /etc/n2n/edge.conf  
```

```conf edge.conf
-d=n2n0 # 指定虚拟网卡名称
-c=com8888 # community名称
-k=888888 # 通讯加密密钥
-a=10.0.34.21 # 在n2n网络中的地址
-l=127.0.0.1:7777 # supernode服务地址
-r # 允许通过n2n转发数据包
```

启动edge

```shell shell
systemctl enable edge
systemctl start edge
```

启动后，可以看到 `n2n` 已经添加了虚拟网卡：

![](edgeVerify.png)

### 开启数据包转发功能

需要通过办公室服务器转发办公室其他电脑到家庭服务器的流量，因此需要在办公室服务器上开启数据包转发功能

需要将 `/etc/sysctl.conf` 文件中的 `net.ipv4.ip_forward` 修改为 1

```bash bash
vi /etc/sysctl.conf
```

```conf sysctl.conf
...
net.ipv4.ip_forward=1
...
```

## 配置办公室网关

### 添加静态路由

其他办公室电脑没有到 `n2n` 网络的路由，因此数据包会发送到办公室网关

此时需要配置办公室网关，添加一条指向 `n2n` 网络的静态路由，下一条为办公室服务器的办公网地址

![](gwRoute.png)

## 配置家庭服务器

### 配置edge，设置自动添加路由

家庭服务器的 `edge` 安装配置过程与办公室服务器的 `edge` 大致相同，但有两点需要注意：

1. 无需添加`-r`参数，因为家庭服务器不需要将来自其他设备的包转发到`n2n`网络
2. 需要添加`-n`参数，这样`edge`启动时会自动产生一条通过`n2n`网络到达办公室网络的路由

```conf edge.conf
-d=n2n0 # 指定虚拟网卡名称
-c=com8888 # community名称
-k=888888 # 通讯加密密钥
-a=10.0.34.41 # 在n2n网络中的地址
-l=88.88.88.88:17777 # supernode公网地址
-n=192.168.34.0/24:10.0.34.21
```

# 验证

## n2n网络验证

使用 `n2n` 网络地址从办公室服务器ping家庭服务器，或从家庭服务器ping办公室服务器，成功

此时数据包的实际流向是 办公室服务器->办公室网关->运营商网络->家庭网关->家庭服务器

由于 `n2n` vpn的配置，此时可以认为办公室服务器和家庭服务器处在同一局域网下，tracert也仅一跳可达

![](trace.png)

## 办公室电脑到家庭服务器网络验证

办公室电脑ping家庭服务器，成功

此时数据包流向是 办公室电脑->办公室网关->办公室服务器->家庭服务器，其中办公室服务器到家庭服务器是 `n2n` 虚拟链路

tracert三跳可达

![](trace2.png)

## 家庭服务器到办公室电脑

家庭服务器ping办公室电脑，成功

此时数据包流向是 家庭服务器->办公室服务器->办公室电脑，其中家庭服务器到办公室服务器是 `n2n` 虚拟链路

tracert两跳可达

![](trace3.png)

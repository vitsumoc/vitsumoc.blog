---
title: 使用SSH包装Socks5代理
url: SSHInSubSocks
date: 2023-11-22 17:38:25
categories:
- 网络编程
tags:
- 网络编程
- SSH
- Go
---

# subSocks简介

[subSocks](https://github.com/luyuhuang/subsocks)是[Luyu Huang](https://luyuhuang.tech/)制作的纯 Go 网络代理软件。

这里是作者本人对此项目的介绍[文档](https://luyuhuang.tech/2020/12/02/subsocks.html)。

<!-- more -->

# 为什么要做SSH包装

因为之前使用v2ray总是被封端口，但是VPS上的22端口始终建在，考虑到SSH协议比较复杂，包括了Shell，SFTP等多种应用。我认为使用SSH协议包装流量可以起到一定的伪装作用，减少端口被封的可能性。

subSocks项目的代码结构非常漂亮，添加SSH包装非常便捷。

# 实现过程

首先需要了解subSocks的代码结构，Luyu Huang的[文档](https://luyuhuang.tech/2020/12/02/subsocks.html)中描述的非常详细，我只需要实现SSHWarpper和SSHStripper。

Go 已经提供了SSH的官方实现，参考[文档](https://pkg.go.dev/golang.org/x/crypto/ssh)。并且提供了使用SSH进行远程Shell的示例。

之后需要对SSH的[通讯过程](/SSH.html)，```Session``` ```Channel``` ```Request```等等各种概念有基础的了解。

使用ssh包中的代码，在服务端使用TCP链接，创建SSH服务器，等待客户端链接后获取Channel，将Channel包装为Stripper。

客户端与服务端相似，需要使用TCP链接，向服务端完成握手过程，之后可获得Session，将Session包装成Wrapper。

# 源码

改动的源码请参考我 fork 的[仓库](https://github.com/vitsumoc/subsocks)

# 使用

为了能够使用ssh协议进行代理，并且能支持密码和密钥认证，配置文件添加了一些字段

### 客户端

```toml client.toml
[client] # client configuration

listen = "127.0.0.1:1080"
username = "subsocks"
password = "subsocks"

server.protocol = "ssh"
server.address = "127.0.0.1:22"

ssh.key = "./ssWithPass"
ssh.passphrase = "123123"
```

- server.protocol：当配置为 ssh 时使用 ssh 协议代理流量
- username，password：这两个字段配置后，ssh协议可以使用账号密码的方式向服务端认证
- ssh.key：配置后客户端可以使用密钥方式向服务端认证，这里配置客户端私钥文件，并把公钥文件存放在服务端
- ssh.passphrase：如果私钥有密码的话，在这里配置
- 账号密码验证和密钥验证是选配的，配置其中一种或两者皆配置都可以。

服务端
```toml server.toml
[server] # server configuration

protocol = "ssh"
listen = "0.0.0.0:22"

ssh.key = "./ssWithPass"
ssh.passphrase = "123123"
ssh.cert = "./ssWithPass.pub"

[server.users]
"subsocks" = "subsocks"
```

- protocol ：当配置为 ssh 时使用 ssh 协议代理流量
- ssh.key：服务端的 ssh 私钥路径，必填
- ssh.passphrase：服务端 ssh 私钥的密码，有密码就填
-  [server.users]：支持的账号密码对，配合客户端的账号密码校验
- ssh.cert： 客户端 ssh 公钥路径，用来支持客户端和服务端之间的密钥认证

# SSH私钥加密的说明

由于 Go 中的ssh暂时还没有支持加密的PKCS#8 format，如果想要使用加密的私钥，需要选择PEM格式。可以用如下命令生成：

ssh-keygen -t rsa -b 4096 -m PEM
使用 ssh-keygen 的默认参数生成的带密码的私钥，在 ssh.ParsePrivateKeyWithPassphrase 过程中会报错。

情况参考：[gaia-pipeline/gaia#182 (comment)](https://github.com/gaia-pipeline/gaia/issues/182#issuecomment-502120516)

# 验证

通过抓包验证，握手过程正常，通讯过程与SSH相同，多条链接使用正常，所有数据均经过加密：

![](wireshark.png)

使用 SSH 代理看了一会视频网站，效果也不错，很流畅。
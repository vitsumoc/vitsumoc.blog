---
title: "[翻译] n2n 常见问题"
url: translate-n2n-faq
date: 2023-12-07 16:55:07
categories:
- 网络工具
tags:
- 网络工具
- 网络编程
- 翻译
---

> 原文 [n2n Frequently Asked Questions](https://github.com/ntop/n2n/blob/dev/doc/Faq.md)

<!-- more -->

# n2n 常见问题

## 发布

### 哪里能找到Windows系统的n2n软件？

我们没有在`release`中发布Windows版本的n2n，但是我们的自动化测试流程会创建他们。你可以点击项目界面中的 `Actions`，选择 `Testing`，进入最近一次的运行实例，在界面最下方的 `Artifacts` 处下载 `binaries`，其中就包括了编译好的windows版本`n2n`。通常来说你可以使用 `x86_64-w64-mingw32\usr\sbin` 路径下的版本。

此外，正如我们的[README](https://github.com/ntop/n2n#further-readings-and-related-projects)中提到的，luckytu 一直在更新`n2n`的[Windows版本](https://github.com/lucktu/n2n)，你也可以在他那里直接下载编译后的软件。

## Supernode

### 我想部署一个私有的，有密码保护的supernode，需要怎么设置？

你可以直接配置 `community.list` 文件，在其中设置一个 `<community name>` (输入单行文本即可) ，然后把这个 `<community name>` 当作您的密码。

在启动 `supernode` 时，记得加上 `-c <community file>` 参数指定配置文件。这样，只有设置了 `-c <community name>` 的 `edge` 可以使用 `supernode`。

此时，在您的 `edge` 向 `supernode` 注册的过程中，`<community name>` 是明文传递的。如果您想要对传输过程进行加密，需要在**所有** `edge` 节点启动时添加 `-H` 参数。

另外，请参阅 `n2n` 附带的 `community.list` 文件以了解该文件的高级使用。

除了这个访问障碍之外，您可能希望在边缘使用有效负载加密 -A_。

只有边缘（而不是超级节点）能够解密有效负载数据。

因此，即使任何人都能够打破超级节点的访问障碍，有效负载仍然受到有效负载加密的保护，请参阅此文档了解详细信息。

除了上述的这些安全手段之外，您还可以在 `edge` 添加 `-A_` 参数来加密传输的数据。数据的加密和解密都是 `edge` 进行的，所以即使是 `supernode` 节点也无法解密数据内容。因此，即使你的 `supernode` 节点被黑客入侵，你的数据内容依然是被加密算法保护的。更多细节可以参考这一篇关于加密的[文档](https://github.com/ntop/n2n/blob/dev/doc/Crypto.md)。

### 我可以在supernode查看接入的edge列表吗？

可以，`supernode` 通过UDP提供了基本的管理接口，默认端口是5645，可以通过 `-t` 参数修改。

只需要发送一个新的行就可以查询当前状态，例如，在 `supernode` **本机**（远程链接不可以）上按下[ENTER]键，然后输入如下命令：

`netcat -u localhost 5645`

### 支持多个supernode节点的部署方式吗？

支持，这篇[文档](https://github.com/ntop/n2n/blob/dev/doc/Federation.md)描述了如何部署多个 `supernode` 节点来提升网络的可用性。

### supernode可以监听多个UDP端口吗？

`supernode` 本身只支持监听一个端口，但是你应该可以通过做NAT的方式将多个端口映射到同一个端口上，例如：

`sudo iptables -t nat -A PREROUTING -i <network interface name> -d <supernode's ip address> -p udp --dport <additional port number> -j REDIRECT --to-ports <regular supernode port number>`

这条命令可以作为 `ExecStartPost=` 添加到 `supernode` 的 `.service` 文件中（不需要加sudo），如果需要映射多个端口，可以多加几行。

### 这个报错是怎么回事 "process_udp dropped a packet with seemingly encrypted header for which no matching community which uses encrypted headers was found"？

这条报错的意思是 `supernode` 收到了一个无法使用的数据包。`supernode` 先将这个包视为一个未加密的包来处理，如果处理失败的话，`supernode` 会**假定**这是一个加密的数据包，之后 `supernode` 会尝试所有可以生成key的 `community` （排除明确没有加密的`community`）。如果任何 `community` 的key都无法解开此数据包，就会产生这条报错。

如果所有 `edge` 的 `-H` 参数配置是相同的（都有 `-H` 或者都没有 `-H` ），并且重启 `supernode` 后依然报错，最大的可能是 `supernode` 或者 `edge` 的版本不一致，导致了数据包格式不一致。

因此，请确保所有 `edge` **和** `supernode` 具有完全相同的版本，例如：最新的 `_dev_` 分支。

## Edge

### 如何查看p2p链接的状态？

`edge` 同样提供了一个本地的UDP管理端口，包括了 _peers_ 这种已经建立的p2p链接，还有 _pending peers_ 这种通过 `supernode` 中转的链接。

`edge` 的默认管理端口号是 5644，可以通过 `-t` 参数修改。可以在**本机**通过此命令查看：

`netcat -u localhost 5644`

发送空行就可以查看链接信息，对于其他的命令行功能，请通过 `help` 查看。

### edge 反复报错 "Authentication error. MAC or IP address already in use or not released yet by supernode"。是什么问题？

Edge 遇到了 n2n 的防欺骗保护。

它可以防止一个边缘的身份（MAC 和 IP 地址）在原始边缘仍然在线时被其他边缘冒充，请参阅一些详细信息。

大多数情况下，有两种情况可以触发此操作：

这是触发了 `n2n` 的防欺骗保护机制，这个机制可以防止已经在线的 `edge` 节点被其他人冒充，这篇[文档](Authentication.md)有更详细的描述。总之，大部分情况下，有两种可能触发这个机制：

你使用的 MAC 地址或 IP 地址已经被使用了，修改这些参数就可以了。

如果一个 `edge` 非正常退出，例如被 `kill -9 ...` 或 `kill -SIGKILL ...`，那么这个 `edge` 可能没有机会通知 `supernode` 取消注册，因此 `supernode` 仍然认为此 `edge` 在线，此时具有相同 MAC 或 IP 地址的注册就不会成功。

`supernode` 记录 `edge` 的超时时间是两分钟，所以可以等待两分钟，或者换不同的 MAC 和 IP 地址注册。

基本上来说，不管是 `CTRL` + `C` 或是 `kill -SIGTERM ...` 或者 `kill -SIGINT ...` 或者 `kill ...` 不带9，都可以正常的结束 `edge`，在管理接口下发 `stop` 命令也可以停止 `edge` ，所以大部分情况下无需使用 `kill -9 ...`。
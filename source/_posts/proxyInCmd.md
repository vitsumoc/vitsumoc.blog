---
title: 通过proxychains-windows在命令行中使用socks5代理
url: proxyInCmd
date: 2024-02-18 22:57:51
categories:
- 网络工具
tags:
- 网络工具
---

工作时有时会需要使用一些带有网络操作命令行工具，这些工具也可能需要通过socks5进行代理，[proxychains-windows](https://github.com/shunf4/proxychains-windows) 为我们提供了直接的解决方案。

<!-- more -->

# 介绍

在proxychains-windows的[介绍文档](https://github.com/shunf4/proxychains-windows/blob/master/README_zh-Hans.md)中摘录如下内容：

> Proxychains.exe 是一个适用于 Win32(Windows) 和 Cygwin 平台的命令行强制代理工具（Proxifier）。它能够截获大多数 Win32 或 Cygwin 程序的 TCP 连接，强制它们通过一个或多个 SOCKS5 代理隧道。
> Proxychains.exe 通过给动态链接的程序注入一个 DLL，对 Ws2_32.dll 的 Winsock 函数挂钩子的方式来将应用程序的连接重定向到 SOCKS5 代理。

> # 工作原理
> - 主程序 Hook CreateProcessW Win32 API 函数调用。
> - 主程序创建按照用户给定的命令行启动子进程。
> - 创建进程后，挂钩后的 CreateProcessW 函数将 Hook DLL 注入到子进程。当子进程被注入后，它也会 Hook 如下的 Win32 API 函数调用：
>   - CreateProcessW，这样每一个后代进程都会被注入；
>   - connect 和 ConnectEx，这样就劫持了 TCP 连接；
>   - GetAddrInfoW 系列函数，这样可以使用 Fake IP 来追踪访问的域名，用于远程 DNS 解析；
>   - 等等。
> - 主程序并不退出，而是作为一个命名管道服务端存在。子进程与主程序通过命名管道交换包括日志、域名等内容在内的数据。主程序实施大多数关于 Fake IP 和子进程是否还存在的簿记工作。
> - 当所有后代进程退出后，主程序退出。
> - 主程序收到一个 SIGINT（Ctrl-C）后，终止所有后代进程。

# 使用

按照文档，最简单的使用方法为：

```bash
proxychains_win32_x64.exe -f proxychains.conf cmd
```

格式为 `可执行程序` `-f` `配置文件路径` `被代理的命令行工具`，使用 `cmd` 作为工具则可以在新开的命令行中手动使用其他命令行工具，均会被代理。

# 快捷方式

通过创建下述快捷方式，可以更方便的使用 proxychains-windows。

```bash
C:\Windows\System32\cmd.exe /k C:"\Program Files\"proxychains_0.6.8_win32_x64\proxychains_win32_x64.exe -f proxychains.conf cmd
```
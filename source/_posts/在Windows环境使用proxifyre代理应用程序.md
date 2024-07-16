---
title: "在 Windows 环境使用 proxifyre 代理应用程序"
url: usingProxifyreInWindows
date: 2024-04-15 09:03:38
categories:
- 网络工具
tags:
- 网络工具
---

> [wiresock/proxifyre 项目链接](https://github.com/wiresock/proxifyre)

<!-- more -->

# 适用场景

proxifyre 适用于，当您已经拥有一个可用的 socks5 代理，但却不了解如何让某个应用程序通过此代理收发包的情况。

# 简介

proxifyre 用于指定一些应用程序的网络流量通过 socks5 代理。

proxifyre 是基于 Windows 包过滤器 [socksify](https://github.com/wiresock/ndisapi/tree/master/examples/cpp/socksify) demo 做的增强，加入了 UDP 能力，简化了配置。

proxifyre 支持被注册为 Windows 服务。

# 依赖

使用 proxifyre 前需要安装两个依赖：

1. Windows Packet Filter ([WinpkFilter](https://github.com/wiresock/ndisapi/releases))
2. Visual Studio Runtime Libraries([Visual Studio 2022 redistributable download page](https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist?view=msvc-170))

# 配置

proxifyre 默认配置文件名为 app-config.json，将配置文件放到 proxifyre 执行目录下会被直接使用，配置文件示例如下：

```json
{
 "logLevel": "None",
 "proxies": [
         {
         "appNames": ["chrome", "C:\\Program Files\\WindowsApps\\ROBLOXCORPORATION.ROBLOX"],
         "socks5ProxyEndpoint": "158.101.205.51:1080",
         "username": "username1",
         "password": "password1",
         "supportedProtocols": ["TCP", "UDP"]
         }
     ]
}
```

- logLevel: 日志等级，可选项为：`None` `Info` `Deb` `All`
- proxies：代理配置列表
- appNames：被代理程序名，采用模糊匹配，例如 `chrome` 会匹配到包含 `chrome` 字符串的所有程序。如果使用路径则可以精确匹配。
- socks5ProxyEndpoint：socks5 服务位置，地址+端口。
- username：如果 socks5 服务用户名，可不填。
- password：socks5 服务密码，可不填。
- supportedProtocols：被代理的网络协议，支持 TCP UDP。

# 使用

完成配置后，直接运行 ProxiFyre.exe 即可使用，日志文件会生成在 `/logs` 路径。

也可以通过下列命令进行服务的注册、启动、停止和卸载，通过 Windows 服务管理：

- ProxiFyre.exe install
- ProxiFyre.exe start
- ProxiFyre.exe stop
- ProxiFyre.exe uninstall

当然也可以通过 nssm 工具将 ProxiFyre.exe 注册为 Windows 服务。
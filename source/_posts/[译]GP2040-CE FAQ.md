---
title: "[译]GP2040-CE FAQ"
url: "[译]GP2040-CE FAQ"
date: 2024-07-30 14:27:46
categories:
- 豆知识
tags:
- 豆知识
- 嵌入式
---

> [GP2040-CE](https://gp2040-ce.info/) 是一个基于树莓派（或其他） [RP 2040](https://www.raspberrypi.com/documentation/microcontrollers/rp2040.html) 微处理器的开源游戏控制器固件项目，支持多种输入模式，适配多个平台。
> 本文译自 GP2040-CE 项目官网，[FAQ](https://gp2040-ce.info/faq/faq-general)  页面。

<!-- more -->

# 常见问题

## 我应该使用哪种输入模式？

这取决于您使用的平台：

- 使用 `XInput Mode` 作为 PC 游戏和第三方主机适配的首选模式
- 在 PS4 或 PS5 中运行 PS4 游戏时，使用 `PS4 Mode`
- 在 PS3 或 PS4 中使用传统模式时，使用 `PS3 Mode`
- 在任天堂 Switch 上使用  `Switch Mode`
- 在 MAME cabinets 或是 PC 音游等场景使用 `Keyboard Mode`

如果您配置了 USB 主机端口、启用了直通功能以及适当的验证设备，则可以在以下情况下使用 GP2040-CE 控制器。

- 在支持 categorized 控制器（例如街机摇杆、赛车方向盘、飞行模拟操纵杆等）的 PS5 系统上的 PS5 游戏上使用 <code>[PS4 Input Mode](https://gp2040-ce.info/web-configurator/menu-pages/settings#additional-ps4-settings)</code>
- 在 Xbox One、Xbox Series X 和 Xbox Series S 上使用 <code>[Xbox One Input Mode](https://gp2040-ce.info/web-configurator/menu-pages/settings#additional-xbox-one-settings)</code>

如果您使用的是经典或迷你主机，则还有其他 USB 输入模式可与这些模拟主机一起使用。

- Sega Genesis/MegaDrive Mini
- NEOGEO Mini
- PC Engine/Turbografx 16 Mini
- EGRET II Mini
- ASTROCITY Mini
- Playstation Classic

## GP2040-CE 是否原生支持 PS5，PS4 或 Xbox Series 主机？

这些主机实现了一些安全措施用来阻止未经授权的控制器接入，破解或绕过这些措施的过程可能涉及到一些法律问题。如果找到用户友好且完全合法的实现方法，例如 <code>[PS4 Input Mode](https://gp2040-ce.info/web-configurator/menu-pages/settings#additional-ps4-settings)</code> 的实现，将来会支持这些主机。

目前通过直通身份验证支持 PS5、Xbox One 和 Xbox Series 主机

- PS5 目前仅支持使用直通身份验证；请参阅 <code>[PS5 Input Mode](https://gp2040-ce.info/web-configurator/menu-pages/settings#additional-ps5-settings)</code>。
- Xbox One 和 Xbox Series 主机仅支持使用直通身份验证；请参阅 <code>[Xbox One Input Mode](https://gp2040-ce.info/web-configurator/menu-pages/settings#additional-xbox-one-settings)</code>。

## 我能在一个设备上使用多个 GP2040-CE 控制器吗？

是的！每个 GP2040-CE 板都被视为一个单独的控制器。但您需要注意，确保同一时间只运行了一个 Web 配置页面。

如果您需要在街机中安装基于 GP2040-CE 的控制板，请查看 [Player Number add-on](https://gp2040-ce.info/add-ons/player-number) 用来强制指定每个玩家的编号。

## GP2040-CE 的输入延迟真的低于 1ms？

是的！如果您的平台支持 1000 Hz USB 轮询，输入延迟将小于 1 毫秒。GP2040-CE 在所有模式下默认配置为 1000 Hz/1 ms 轮询，但某些系统会覆盖或忽略控制器请求的轮询速率。1000 Hz 轮询率已确认适用于 PC 和 MiSTer。即使您的平台不支持高速 USB 轮询，GP2040-CE 仍会以目标系统允许的最大速度读取和处理您的输入。

## RGB LED、玩家 LED 和 OLED 显示屏等附加功能是否会影响性能？

完全不会！Pico 的 RP2040 处理器有两个内核，GP2040-CE 将其中一个核心专门用于读取、处理和发送玩家输入，所有辅助功能（例如 LED 和显示器）都在辅助核心上运行。无论集成了多少功能，GP2040-CE 都不会引入额外的输入延迟。

## 为什么按键会使用 B3，A1，S2 这种奇怪的标签？

GP2040-CE 使用通用系统来处理按钮输入，类似于带有一些额外按钮的传统 Play Station 控制器布局。

- 4 方向键 (B1-B4)
- 4 肩键 (L1，L2，R1，R2)
- 选择和启动（S1，S2）摇杆按下（L3，R3）
- 两个辅助键（A1，A2）用于引导、PS键、触摸屏、Home 键或者截图键等

GP2040-CE 文档和 Web 配置器都提供了一个下拉菜单，用于将按钮标签更改为更熟悉的控制器布局。您可以参考 [GP2040-CE 说明书](https://gp2040-ce.info/usage#buttons) 上的按键映射表。

# 技术问题

## 内建的 Web 配置页面是什么魔术？

这里没有什么魔法，只是一些很酷的库一起工作：

- 使用 React 和 Bootstrap 的单页应用程序嵌入在 GP2040-CE 固件中
- TinyUSB 库通过 RNDIS 提供虚拟网络连接
- lwIP 库提供了一个 HTTP 服务器，为嵌入式 React 应用程序和 Web 配置 API 提供服务
- ArduinoJson 库用于Web API请求的序列化和反序列化

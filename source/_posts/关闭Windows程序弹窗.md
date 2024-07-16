---
title: 关闭Windows程序弹窗
url: 关闭Windows程序弹窗
date: 2024-06-18 17:22:21
categories:
- Windows
tags:
- Windows
---

> 本文记录了一个使用工具寻找弹窗、分析弹窗、修改程序的过程，对于理解 Windows 编程有一些作用
> 本文中的内容全部来自于此 [视频](https://www.bilibili.com/video/BV1YM4m1m7qc)

<!-- more -->

# 通过 spy++ 获得弹窗的窗口类

spy++ 随着 Visual Studio 一同安装，在工具菜单中。

通过其中的窗口搜索功能，可以搜索到弹出窗口的窗口名称和窗口类，例如在我的 WinRAR 中：

广告弹窗可以看到窗口类为 RarReminder

- 句柄：001E0A8C
- 窗口名称：WinRAR 6.24b1 Expired Notification | Your Free Trial Period has Ended!
- 窗口类：RarReminder

注册弹窗是 Dialog，

- 句柄：007A0950
- 窗口名称：Please purchase WinRAR license
- 窗口类：#32770 (对话框)

# 通过 apimonitor 截取软件对系统 api 的调用

从 [这里](http://www.rohitab.com/) 下载工具。

查找并监控 CreateWindow 相关函数，Dialog 相关函数。

选择被监控程序 WinRAR 并启动，可以看到程序调用 api 的所有记录。

广告弹窗调用类为：

- RarReminder

注册弹窗的调用为：

- DialogBoxParamW ( 0x00007ff643070000, "REMINDER", 0x0000000000600b84, 0x00007ff64316d600, 0 )

查看堆栈，记录顶部偏移地址，分别为：

- 0xbf3cc
- 0xbf46d

# 使用 IDA Pro 修改对应位置的机器码

使用 IDA Pro 查找对应位置的机器码，应该是创建窗口或弹窗的内容。

使用十六进制编辑器，编辑可执行文件，将此处的机器码全部修改为 0x90 （空转）

---
title: 在Go中使用lua
url: Go
date: 2023-12-05 11:25:15
categories:
- 小玩具
tags:
- 小玩具
- Go
- 库
---

使用```gopher-lua```，在 Go 中使用lua。```gopher-lua``` 项目地址：

https://github.com/yuin/gopher-lua

<!-- more -->

使用示例仓库地址：

https://github.com/vitsumoc/my-golua

示例列表：

+ 最基础的用法
+ 基础数据类型
+ 在lua中调用go方法
+ 在go中使用lua协程
+ 示范如何手动开启模块
+ 在lua中使用go模块
+ 在 Go 中调用lua方法
+ 在lua中使用 Go 数据
+ 通过context控制停止
+ 在有协程的情况下使用context控制
+ 共享lua文件字节码, 减少开销
+ 通过go协程跑lua的示例 可以把ch带到lua中 和相关限制
+ 在lua中使用ch的例子
+ lua虚拟机池
+ 在 Go 中提供钩子, 使lua可以注册脚本, 在脚本中获得并修改用户数据
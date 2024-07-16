---
title: "[问题]Windows环境下nssm注册的mysql服务无法启动"
url: question-mysqld-service
date: 2023-12-15 09:30:00
categories:
- 问题
tags:
- 问题
- 项目实践
- Windows
---

# 环境

手上有个项目上一直使用的一键安装包，包括了上位机、后端、前端、数据库、时序库、nginx等一系列东西。一直都是通过 `nssm` 将这些软件注册成自启动服务的。注册的方式大概是这样：

<!-- more -->

```bat install.bat
:: 注册mysql
nssm-2.24\win64\nssm install xxx-scada-mysql %cd%\mysql-8.0.27-winx64\bin\mysqld.exe
nssm-2.24\win64\nssm set xxx-scada-mysql AppDirectory %cd%\mysql-8.0.27-winx64\bin
:: 启动mysql
nssm-2.24\win64\nssm start xxx-scada-mysql
```

前两天我们需要在公司的一台测试服务器上安装这套项目软件，先检查了公司的服务器环境，发现已经有了 `mysql` 和 `nginx` 服务，于是手动停止这两个服务，之后使用一键安装包部署项目。

> 此时，系统中有一个之前已经安装的 `mysql`，称为 `数据库A`。`数据库A` 通过 `mysqld install` 命令安装了服务，称为 `服务A`， `服务A` 已经被手动停止运行。
> 一键安装包中又拷贝了一份 `mysql` 进去，称为 `数据库B`。通过 `nssm` 安装的 `数据库B` 服务称为 `服务B`。

# 问题过程

1. 发现通过 `nssm` 注册的 `服务B` 无法启动，所以关闭 `服务B`。
2. 手动运行 `数据库B` 中的 `mysqld` 程序，发现程序闪退，没有报错信息，也没有错误日志。
3. 怀疑是依赖问题，尝试了更新 `MSVC` ，没有效果。
4. 尝试使用 `数据库B` 中的 `mysqld --log-error=my.err` ，发现 `mysqld` 不再闪退，但是此时依然不能正常提供数据库服务，并且没有异常的错误日志。
5. 同事启动了 `服务A` ，发现可以正常使用。
6. 受同事启发，尝试删除 `服务A`，此时脑袋混乱，居然是使用 `数据库B` 执行的 `mysqld --remove`，没想到依然能删除 `服务A`。
7. 发现删除 `服务A` 后，`数据库B` 中的 `mysqld` 可以正常使用了，再次尝试 `服务B` ，发现也可以正常使用。

# 思考

本次问题的出现，主要原因还是我对 `Windows` 系统不熟悉，对于 `Windows` 系统中服务注册原理完全不懂。

长期使用 `nssm` 进行服务管理，让我们可以一直忽略 `Windows` 的服务管理细节，不断地向前走下去。同时也让我们失去了探索 `Windows` 服务管理的动力。其实，假如世界上没有 `nssm` ，也许需要一周，也许需要一两个月，我们总是能学会注册服务的方法。

因为工具过于方便导致失去了底层能力，这次的问题只是这个道理的再一次体现而已。
---
title: 在Linux中使用Systemd管理自启动
url: 在Linux中使用Systemd管理自启动
date: 2023-12-01 14:19:42
categories:
- 豆知识
tags:
- 豆知识
- 环境配置
- Linux
---

# 常用命令

启动服务

```shell shell
systemctl start service-name
```

<!-- more -->

停止服务

```shell shell
systemctl stop service-name
```

查看服务状态

```shell shell
systemctl status service-name
```

设置开机自启动

```shell shell
systemctl enable service-name
```

停止开机自启动

```shell shell
systemctl disable service-name
```

# 服务注册

在 ```/etc/systemd/system``` 路径下，创建 ```service-name.service``` 文件，格式如下：

```ini service-name.service
[Unit]
# 服务名称
Description = xxxx server
After = network.target syslog.target
Wants = network.target

[Service]
Type = simple
# 启动命令
ExecStart = /home/start.sh

[Install]
WantedBy = multi-user.target
```
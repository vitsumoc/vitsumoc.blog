---
title: 简单的Linux服务管理
url: 简单的Linux服务管理
date: 2024-05-22 15:30:38
categories:
- 运维
tags:
- 运维
- Linux
---

在这里记录一些基础的 `Linux` 服务管理维护方式，作为程序员，总是会用到的。

<!-- more -->

# 手动运行

以一个 Java Web 项目为例，手动运行 Jar 包可能是这样的：

```bash
nohup java -jar YongWang-1.0-SNAPSHOT.jar > /dev/null 2>&1 &
```

这样程序就可以在后台运行，但是有一些缺点：

- 系统重启后程序不会启动
- 只能通过 `ps` 查询进程状态
- 想要关闭程序的话，需要通过 `ps` 查找进程号，然后手动 `kill`
- 难以接入各种自动化工具

因此，将程序注册为服务是一个更好的选择。

# 注册服务

如果想将我们的程序注册为服务，首先需要编写一个配置文件，这个文件可以放在 `/etc/systemd/system/` 路径下。

文件的名称一般参考服务的名称，并以 `.server` 结尾，因此在这个例子中，我的文件名为 `yongwang.service`：

```text yongwang.service
[Unit]
Description=YongWang Server

[Service]
WorkingDirectory=/home/vc/ywyl/server/
ExecStart=java -jar YongWang-1.0-SNAPSHOT.jar

[Install]
WantedBy=multi-user.target
```

之后，需要通过命令注册服务

```bash
systemctl daemon-reload
```

# 服务管理命令

后续我们对服务的管理，一般包括：

- 启动服务

```bash
systemctl start yongwang
```

- 停止服务

```bash
systemctl stop yongwang
```

- 查看服务状态

```bash
systemctl status yongwang
```

- 设置开机自启

```bash
systemctl enable yongwang
```

- 取消开机自启

```bash
systemctl disable yongwang
```
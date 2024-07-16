---
title: "mysql bin_log 塞满了磁盘"
url: "mysql bin_log 塞满了磁盘"
date: 2024-06-26 10:39:35
categories:
- 运维
tags:
- 运维
- 项目实践
---

项目制的开发就是这样，你能从中获得一些宝贵的一手经验，但代价是每一次经验背后都有一个“事故”。

<!-- more -->

# 问题

客户告诉我，服务器磁盘增长速度异常，项目上线二十天，mysql 文件夹的磁盘占用就超过 700G。

*非常非常感谢这位能够提前帮我们发现问题的客户，责任心和专业性都很强。*

经检查，是 mysql/data 文件夹中出现了大量的 binlog.xxxxxx 文件，每个文件大小大约 1GB，每天产生若干个文件。

查阅资料后，发现该文件是 mysql 的二进制日志，可以用来进行数据恢复、数据备份等操作。（在我这个项目中用不到，另有备份机制）。

# 处理

解决方案是这样的：

1. 在 mysql 命令行中查找 log_bin 相关的配置 `show variables like '%log_bin%';`

发现输出包括 `log_bin ON`，表示二进制日志功能已经被打开。

2. 通过 `PURGE BINARY LOGS BEFORE '2024-06-26 10:00:00';` 删除原有的日志文件

**必须在日志功能开启的情况下执行上述命令删除文件，否则命令无效**

3. 在 mysql 启动脚本中添加参数 `--disable-log-bin`，重启 mysql 服务，再次检查 log_bin 配置

发现输出包括 `log_bin OFF`，表示二进制日志功能已经被关闭，后续不再产生日志。

# 参考文档

- [Binary Logging Options and Variables](https://dev.mysql.com/doc/refman/8.4/en/replication-options-binary-log.html)
- [MySQL的binlog日志详解](https://www.cnblogs.com/luckly-hf/p/14324957.html)
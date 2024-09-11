---
title: 在Ubuntu上安装SVN服务
url: 在Ubuntu上安装SVN服务
date: 2024-09-11 13:22:06
categories:
- 运维
tags:
- 运维
---

[参考](https://www.cnblogs.com/xiaostudy/p/11374100.html)

### 安装 SVN 服务

```bash
sudo apt-get install subversion
```

<!-- more -->

### 创建根目录文件夹

```bash
mkdir /root/svn
```

### 创建仓库文件夹

```bash
mkdir /root/svn/shidian
```

### 创建仓库

```bash
svnadmin create /root/svn/shidian
```

### 仓库创建成功

文件夹中生成了下列内容：
- conf
- db
- format
- hooks
- locks
- README.txt

### 启动服务

svnserve -d -r /root/svn --listen-port 17749

### 创建全局的账号密码和权限文件

```bash
touch /root/svn/passwd
touch /root/svn/authz
```

### 添加账号密码

```bash
vi /root/svn/passwd
```

```text passwd
[users]
user1 = password
```

### 添加权限配置

```bash
vi /root/svn/authz
```

可按分组或按用户配置：

```text authz
[groups]
coder = user1,user2

[shidian:/]
@coder = rw
* = r

[u1:/]
user1 = rw
```

### 在仓库中配置鉴权

```bash
vi /root/svn/shidian/conf/svnserve.conf
```

修改下列四行

```conf svnserve.conf
anon-access = none
auth-access = write
password-db = /root/svn/passwd
authz-db = /root/svn/authz
```
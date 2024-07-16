---
title: 使用Jenkins自动部署项目
url: 使用Jenkins自动部署项目
date: 2024-05-17 15:09:35
categories:
- 运维
tags:
- 运维
---

# 前言

笔者近期遭遇了一些运维上的困扰。

<!-- more -->

当项目实例部署到很多站点以后，每一次想要更新项目都需要远程连接进入服务器，进行一系列繁琐的运维操作，这样很耽误时间，而且容易犯错，当服务器网络环境不好的时候更是让人非常狂躁。

当时就想做一个简易的部署工具来解决这个问题，后来又意识到这应该是一个普遍的需求，很可能已经拥有了成熟的工具，向朋友打听后，朋友推荐了 jenkins。

# 基本概念

jenkins 是一个自动化运维工具，在业内应该是被称为 `持续集成(Continuous integration)` 工具，还有一些什么持续部署，持续交付之类的概念，总之就是一套把程序员的代码变成产品交付给客户的东西。

jenkins 是一个 JavaWeb 程序，支持非常丰富的插件，而且绝大部分 `动作` 都是由插件来完成，对了，`动作` 是我随便用的词，总之就是我们手动部署中的每一个最小化步骤，例如拉代码、打包、上传，我都称为 `动作`。

jenkins 里有 `项目(Project)` 的概念，其实就是一个完整的项目部署过程啦，显而易见的，项目是顺序 `动作` 的集合。

jenkins 中的 `项目` 可以被 `执行(Build)`，`执行` 一般就是从代码仓库拉代码、构建项目、部署项目这样三个大的步骤（当然是自动的），每一次 `执行` 都会有自己的执行结果，执行日志。

jenkins 中很重要的部分是 `插件(plugin)`，`插件` 也是由全世界各地的开发者开发的，很多 `动作` `环境` 甚至 `项目` 还有很多其他各种各样的东西，都是由 `插件` 提供的，因此当你想使用 jenkins 进行某事情的时候，往往会以 装个插件 作为起点。

# 使用案例

### 拉 SVN 代码，通过 npm 打包，部署到 windows 服务器

这个例子是一个前端项目，代码存放在 SVN，通过 npm 打包成 dist 文件夹，之后部署到境外的 Windows 服务器上。

这个例子里，jenkins 直接被安装在了目标服务器上，因此部署操作只需替换文件夹即可。

#### svn 相关

- 安装 Subversion Plug-in 插件
- 全局配置可用账号密码
- 在项目中配置代码仓库

#### npm 打包相关

- 安装 Node 插件
- 全局添加 Node 版本环境
- 在项目中选择 Node 环境
- 配置打包指令 `npm -g install` `npm run build`
- 对于麻烦的 npm 依赖问题，可以直接把开发环境的 node_modules 拷贝到 jenkins 的工作目录

#### 部署

- 删除原文件夹的内容 `del /f /s /q D:\arctech-3.1-pak\server\arctech\dist 1>nul`
- 删除原文件夹 `rd /s /q D:\arctech-3.1-pak\server\arctech\dist`(老版本的windowsServer系统可能有点问题，必须先删内容后删文件夹，参考 [这里](https://stackoverflow.com/questions/22948189/how-to-solve-the-directory-is-not-empty-error-when-running-rmdir-command-in-a))
- 复制新文件夹 `echo d | xcopy dist D:\arctech-3.1-pak\server\arctech\dist /E`

### 拉 SVN 代码，使用 maven 打 jar 包，部署到 windows 服务器

从 SVN 拉代码，使用 maven 构建，部署到本机 windows 系统。

拉代码步骤略过。

#### maven相关

- 安装 Maven Integration plugin 插件
- 配置 pom 文件路径
- 配置 mvn 命令 `install`
- 配置 Post Steps，仅当 maven 构建成功时执行后续的步骤

#### 部署

- 停用服务 `net stop arctech-scada-server`
- 替换文件 `copy /Y target\ZXBoPlatform-3.1-SNAPSHOT.jar D:\vc\arctech-3.0-pak\server\ZXBoPlatform\ZXBoPlatform-3.1-SNAPSHOT.jar`
- 启动服务 `net start arctech-scada-server`

### 通过 SSH 部署到 Linux 服务器

通过 SVN 拉代码，使用 maven 构建，部署到远端 Linux 系统。

拉代码和构建步骤略过。

#### 通过SSH部署

- 安装 SSH 部署插件 `Publish Over SSH`
- 在 jenkins system 下，配置 SSH 服务器
- 在 maven 项目的构建后步骤中，添加 `Send build artifacts over SSH`
- 设置源文件 `target/YongWang-1.0-SNAPSHOT.jar`
- 设置目标路径 `/ywyl/server_temp`
- 设置部署命令 `mv /home/vc/ywyl/server_temp/target/YongWang-1.0-SNAPSHOT.jar /home/vc/ywyl/server/YongWang-1.0-SNAPSHOT.jar && systemctl restart yongwang`

# 总结

将打包部署工作自动化以后，对于笔者而言获得了这些收益：

1. 省去了打包，部署的时间
2. 降低了项目部署中犯错的可能性
3. 对于网络环境较差的服务器，提升了部署的成功率
4. 能够将多个项目实例集中管理起来
5. 让大家更爱提交代码

# 参考

- [jenkins](https://www.jenkins.io/)
- [通过SVN钩子，实现提交触发部署](https://blog.csdn.net/qq_54796785/article/details/133899251)
- [使用maven插件构建maven项目](https://blog.csdn.net/sixeleven611/article/details/119664695)
- [使用SSH插件，向目标服务器部署](https://blog.csdn.net/liaomingwu/article/details/123483289)
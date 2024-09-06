---
title: Caddy常用配置示例
url: Caddy常用配置示例
date: 2024-08-27 09:31:29
categories:
- 运维
tags:
- 运维
- caddy
- 项目实践
---

# Caddy常用配置示例

最近将公司部分业务从 `nginx` 迁移到 `caddy`，为了避免翻车，总结了几个最常见的用法

<!-- more -->

## 端口固定响应

- 用来测试运维同事有没有正确开放 `7000` 端口。

```caddy
:7000 {
  respond "Hello, im 7000！"
}
```

## 代理静态站点和后端服务

- 前端页面存放在 `/home/lenovo/vc/ywyl/web`
- 后台服务接口格式 `http://127.0.0.1/m/user/info`

```caddy
:7000 {
  # 静态站点根目录
  root * /home/lenovo/vc/ywyl/web
  # 后端服务, 真实地址是 http://127.0.0.1:9090/m/user/info
	handle /m/* {
		reverse_proxy http://127.0.0.1:9090
	}
  # 静态站点服务
	handle {
		file_server
	}
}
```

## 自定义前缀用来区分业务

- 将 `http://localhost:7000/ecard/info` 代理到 `http://ipServer:80/info`

```caddy
:7000 {
  handle_path /ecard/* {
	  reverse_proxy http://ipServer:80
  }
}
```

## 自定义前缀, 匹配一个有后缀的服务

- 将 `http://127.0.0.1:8090/test2/tt2` 代理到 `http://127.0.0.1:8080/tt1/tt2`

```caddy
:8090 {
  # 8090/test2/tt2 -> 8080/tt1/tt2
  handle_path /test2/* {
    rewrite * /tt1{uri}
	  reverse_proxy http://127.0.0.1:8080
  }
}
```

## 后端服务是 HTTPS, 或是会校验 Host 头的情况

- 需要求改请求头, 否则请求不成功

```caddy
example.com {
	reverse_proxy https://example.com {
		header_up Host {upstream_hostport}
	}
}
```

## 添加前缀, 代理某后端的 GET 请求, 处理路径问题

- `:8090/comm/?p=aaa` -> `https://example.com/index.php?p=aaa`
- 此处可以使用 `uri[1:]` 语法手动处理 `url`

```caddy
:8090 {
  handle_path /comm/* {
  	rewrite * /index.php{uri[1:]}
  	reverse_proxy https://example.com {
  		header_up Host {upstream_hostport}
  	}
  }
}
```

# 一些排查问题的方法

使用 `apt install` 安装后，caddy 的服务配置文件默认放置在 `/lib/systemd/system`。

`caddy` 会默认创建一个 `caddy` 用户用来执行服务，并将此用户的 `$HOME` 设置为 `/var/lib/caddy`，这意味着相关的证书、自动存储的配置文件会被保存在这里。

如果是排查和证书相关的问题，使用 `root` 用户和 `caddy` 用户的环境可能不同，此时可以切换到 `caddy` 用户检查：

先使用 `root` 授权，让普通用户也可以使用 `caddy` 程序占用 `80` 和 `443` 端口：

```bash
setcap 'cap_net_bind_service=+ep' /usr/bin/caddy
```

再进入 `caddy` 用户，手动运行程序，记得在配置文件前先打开 `debug` 显示：

```bash
su -s /bin/bash caddy
cd /etc/caddy
caddy run -c Caddyfile
```
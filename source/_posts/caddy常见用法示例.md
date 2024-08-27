---
title: Caddy常用配置示例
url: Caddy常用配置示例
date: 2024-08-27 09:31:29
categories:
- caddy
tags:
- caddy
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
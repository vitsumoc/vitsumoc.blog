---
title: 内网项目添加HTTPS支持
url: 内网项目添加HTTPS支持
date: 2024-10-22 10:43:29
categories:
- HTTPS
tags:
- HTTPS
- 运维
- 项目实践
- caddy
---

公司实施了一些部署在客户内网的项目，思来想去还是用 HTTPS 访问服务器比较好，既解决了很多数据安全的问题，又满足了很多浏览器功能的安全性要求。

<!-- more -->

实现路径大致如下：

- 创建和签名SSL/TLS证书
- 部署 HTTPS 服务
- 修改 DNS 或客户端 host，让客户端可将域名解析到服务器
- 客户端安装证书

# 创建和签名 SSL/TLS 证书

> [参考](https://www.cnblogs.com/shisuizhe/p/13712591.html)
> [参考](https://monkeywie.cn/2019/11/15/create-ssl-cert-with-san/)

- CA私钥和自签证书
```bash
openssl req -x509 -newkey rsa:4096 -days 25568 -keyout ca-key.pem -out ca-cert.pem -subj "/C=cn/ST=shenzhen/L=shenzhen/O=msj/OU=msj/CN=msj"
Enter PEM pass phrase:******
```

- 服务器私钥和请求
```bash
openssl req -newkey rsa:4096 -nodes -keyout battery-cap-key.pem -out battery-cap-req.pem -subj "/C=cn/ST=shenzhen/L=shenzhen/O=msj/OU=msj/CN=battery-cap.msj" -reqexts SAN -config openssl.cnf
```

- CA 签署服务器证书
```bash
openssl ca -in battery-cap-req.pem -md sha256 -days 25568 -keyfile ca-key.pem -cert ca-cert.pem -extensions SAN -config openssl.cnf -out battery-cap-cert.pem
```

# 部署 HTTPS 服务

> [参考](/Caddy常用配置示例.html)

可以先部署静态站点进行测试

```caddy
battery-cap.msj {
	encode zstd gzip
	tls /home/caddy/herong/tls/battery-cap-cert.pem /home/caddy/herong/tls/battery-cap-key.pem
	handle {
		root * /home/caddy/herong/web/dist
		file_server
	}
}
```

# 手动修改host

C:\Windows\System32\drivers\etc\hosts

```
192.168.34.197 battery-cap.msj
```

# 客户端手动安装证书

[参考](https://wenku.csdn.net/answer/eecfaab1adcf40a3a5e8724c4e3454fe)

# 自动修改host、安装证书

通过一个程序将修改 host 文件和安装证书两件事情放到一起，方便实施同事交付。

```go
func main() {
	domain := "battery-cap.msj"
	address := "192.168.34.197"
	cert := "msj-cert.pem"

	// 日志
	file, err := os.Create("证书安装.log")
	if err != nil {
		MessageBoxPlain("提示", "失败")
		log.Fatal(err)
	}
	defer file.Close()
	logger := log.New(file, "", log.LstdFlags)

	// 参数回显
	logger.Println("域名：" + domain)
	logger.Println("地址：" + address)
	logger.Println("证书：" + cert)

	// 改host
	logger.Println("修改 host 文件")
	hostFile, err := os.OpenFile("C:/Windows/System32/drivers/etc/hosts", os.O_WRONLY|os.O_APPEND, 0600)
	if err != nil {
		MessageBoxPlain("提示", "失败")
		logger.Fatal(err)
		return
	}
	defer hostFile.Close()
	hostFile.WriteString(address + " " + domain + "\n")
	logger.Println("ok")

	// 安装证书
	logger.Println("安装证书")
	cmd := exec.Command("cmd", "/C", "certmgr.exe /c /add  "+cert+" /s root")
	output, err := cmd.CombinedOutput()
	if err != nil {
		MessageBoxPlain("提示", "失败")
		logger.Fatal(err)
	}
	logger.Println(output)
	logger.Println("ok")

	MessageBoxPlain("提示", "完成")
}
```
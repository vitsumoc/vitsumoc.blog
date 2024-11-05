---
title: "[问题]在NAT背后使用Caddy提供HTTPS服务"
url: "[问题]在NAT背后使用Caddy提供HTTPS服务"
date: 2024-10-28 14:02:49
categories:
- HTTPS
tags:
- HTTPS
- 运维
- 项目实践
- caddy
---

前段时间，我尝试了 [在内网中使用 HTTPS 部署](/内网项目添加HTTPS支持)，但立刻就在随后的实践中发现了一个问题。

<!-- more -->

我的领导告诉我，客户并不想运行我的证书安装程序，客户只是想打开浏览器输入地址就可以访问系统（因为这样比较方便）。

于是我就使用服务器的内网地址签发了一张证书，这样在内网通过 `https://ip` 的方式就可以访问系统了，虽然浏览器会提示不安全，但是所有功能也都可以正常使用，问题就算解决了。

随后，我想把这个服务通过一台路由设备代理到外网，也通过 https 访问，这样我可以方便的进行一些运维测试之类的工作。我就这样掉进了坑里。

# 环境

涉及到的设备包括：

- 公网客户端（有公网地址的普通电脑一台）
- 路由器（用来做 NAT，只有基本的 NAT 能力）
- 服务器（在内网提供服务）

涉及到的网络地址包括：

- 路由器公网地址：36.33.xx.xx
- 路由器内网地址：192.168.34.1
- 服务器内网地址：192.168.34.197

当前已经可以在内网使用 `https://192.168.34.197` 访问服务，目标是通过配置路由器 NAT 和服务器上的 caddy，使公网客户端可以使用 `https://36.33.xx.xx` 访问服务。

*公网地址的 443 端口确定可用，之前测试过*

# 失败的尝试

### 公网地址 443 端口代理到服务器 443

最开始的思路比较简单，直接配置路由器将公网 443 端口映射到服务器 443 端口。

Caddyfile的配置也很简单：

```caddy
https://192.168.34.197 {
	encode zstd gzip
	tls /home/caddy/herong/tls/19216834197-cert.pem /home/caddy/herong/tls/19216834197-key.pem
        handle {
                root * /home/caddy/herong/web/dist
                file_server
        }
}
```

这种情况下，浏览器在报安全验证之后就可以进入网页，但请求不返回任何内容。观察 `response` 发现：

- content-length: 0
- server: Caddy

服务到达了 caddy，但是没有提供网站内容，接下来查看 caddy 的日志发现：

```log
2024/10/30 07:05:56.420	DEBUG	http.stdlib	http: TLS handshake error from 36.33.xx.xx:65222: remote error: tls: unknown certificate
2024/10/30 07:05:56.422	DEBUG	events	event	{"name": "tls_get_certificate", "id": "9a9f30ec-e627-48e4-b7ee-52beb6bad236", "origin": "tls", "data": {"client_hello":{"CipherSuites":[10794,4865,4866,4867,49195,49199,49196,49200,52393,52392,49171,49172,156,157,47,53],"ServerName":"","SupportedCurves":[64250,25497,29,23,24],"SupportedPoints":"AA==","SignatureSchemes":[1027,2052,1025,1283,2053,1281,2054,1537],"SupportedProtos":["h2","http/1.1"],"SupportedVersions":[60138,772,771],"RemoteAddr":{"IP":"36.33.xx.xx","Port":65223,"Zone":""},"LocalAddr":{"IP":"192.168.34.197","Port":443,"Zone":""}}}}
2024/10/30 07:05:56.422	DEBUG	tls.handshake	choosing certificate	{"identifier": "192.168.34.197", "num_choices": 1}
2024/10/30 07:05:56.422	DEBUG	tls.handshake	default certificate selection results	{"identifier": "192.168.34.197", "subjects": ["192.168.34.197", "*.192.168.34.197"], "managed": false, "issuer_key": "", "hash": "00c8e2e97b167d3f278fe71d0a962cd29174fec485fe2c00f25bb718e345f961"}
2024/10/30 07:05:56.422	DEBUG	tls.handshake	matched certificate in cache	{"remote_ip": "36.33.xx.xx", "remote_port": "65223", "subjects": ["192.168.34.197", "*.192.168.34.197"], "managed": false, "expiration": "2094/10/31 03:05:40.000", "hash": "00c8e2e97b167d3f278fe71d0a962cd29174fec485fe2c00f25bb718e345f961"}
```

从日志里可以看到 `remote error: tls: unknown certificate`，我猜想是客户端不认证书，然后看到证书的 subjects 是 `192.168.34.197`，我就觉得是因为客户端输入的地址和证书提供的 CN 或者 SAN 不匹配导致的问题。

然后我就觉得，如果服务器能够提供 `36.33.xx.xx` 的证书就好了。

### 使用公网地址的证书

然后就按照公网地址制作了证书，修改 Caddyfile：

```caddy
https://192.168.34.197 {
	encode zstd gzip
	tls /home/caddy/herong/tls/3633xxxx-cert.pem /home/caddy/herong/tls/3633xxxx-key.pem
        handle {
                root * /home/caddy/herong/web/dist
                file_server
        }
}
```

这下连 443 端口都没起来，查看日志是这样的：

```log
2024/10/30 07:16:40.600	WARN	tls	stapling OCSP	{"error": "no OCSP stapling for [36.33.xx.xx *.36.33.xx.xx]: no OCSP server specified in certificate"}
2024/10/30 07:16:40.600	DEBUG	events	event	{"name": "cached_unmanaged_cert", "id": "ca1f1002-a94b-4af4-a7fe-a88960129791", "origin": "tls", "data": {"sans":["36.33.xx.xx","*.36.33.xx.xx"]}}
2024/10/30 07:16:40.600	DEBUG	tls.cache	added certificate to cache	{"subjects": ["36.33.xx.xx", "*.36.33.xx.xx"], "expiration": "2094/10/31 03:01:19.000", "managed": false, "issuer_key": "", "hash": "15f7c42ab20c2daa817947d6d066a353fea00e6e69e2c873c33d3ebd2542257a", "cache_size": 1, "cache_capacity": 10000}
2024/10/30 07:16:40.600	INFO	http.auto_https	enabling automatic HTTP->HTTPS redirects	{"server_name": "srv0"}
2024/10/30 07:16:40.600	DEBUG	http.auto_https	adjusted config	{"tls": {"automation":{"policies":[{"subjects":["192.168.34.197"]},{}]}}, "http": {"servers":{"remaining_auto_https_redirects":{"listen":[":80"],"routes":[{},{}]},"srv0":{"listen":[":443"],"routes":[{"handle":[{"handler":"subroute","routes":[{"handle":[{"encodings":{"gzip":{},"zstd":{}},"handler":"encode","prefer":["zstd","gzip"]},{"handler":"subroute","routes":[{"handle":[{"handler":"vars","root":"/home/caddy/herong/web/dist"},{"handler":"file_server","hide":["./Caddyfile"]}]}]}]}]}],"terminal":true}],"tls_connection_policies":[{"match":{"sni":["192.168.34.197"]},"certificate_selection":{"any_tag":["cert0"]}},{}],"automatic_https":{}}}}}
```

怀疑是证书中的地址和 caddy 配置中的地址对不上。

于是就再签了一张 SAN 同时包括两个地址的证书。

### 同时包含公网和私网的证书

制作了 CN 为 `36.33.xx.xx`，SAN 同时包括 `36.33.xx.xx` 和 `192.168.34.197` 的证书，使用后 443 端口算是能够起来了，caddy 配置文件也只改了证书：

```caddy
https://192.168.34.197 {
	encode zstd gzip
	tls /home/caddy/herong/tls/3633xxxxw197-cert.pem /home/caddy/herong/tls/3633xxxxw197-key.pem
        handle {
                root * /home/caddy/herong/web/dist
                file_server
        }
}
```

但是！客户端访问的问题依然存在，问题现象也和之前一模一样。只不过这次从客户端看到的证书的 CN 和 SAN 都是 `36.33.xx.xx`，这一点倒是符合预期。

查看 caddy 的日志，发现连报错都一样：

```log
2024/10/30 07:25:45.013	DEBUG	http.stdlib	http: TLS handshake error from 36.33.xx.xx:49913: remote error: tls: unknown certificate
2024/10/30 07:25:45.015	DEBUG	events	event	{"name": "tls_get_certificate", "id": "aa9a98f8-1058-4331-a1d9-e642f0645cf4", "origin": "tls", "data": {"client_hello":{"CipherSuites":[19018,4865,4866,4867,49195,49199,49196,49200,52393,52392,49171,49172,156,157,47,53],"ServerName":"","SupportedCurves":[60138,25497,29,23,24],"SupportedPoints":"AA==","SignatureSchemes":[1027,2052,1025,1283,2053,1281,2054,1537],"SupportedProtos":["h2","http/1.1"],"SupportedVersions":[51914,772,771],"RemoteAddr":{"IP":"36.33.xx.xx","Port":49914,"Zone":""},"LocalAddr":{"IP":"192.168.34.197","Port":443,"Zone":""}}}}
2024/10/30 07:25:45.015	DEBUG	tls.handshake	choosing certificate	{"identifier": "192.168.34.197", "num_choices": 1}
2024/10/30 07:25:45.015	DEBUG	tls.handshake	default certificate selection results	{"identifier": "192.168.34.197", "subjects": ["36.33.xx.xx", "192.168.34.197"], "managed": false, "issuer_key": "", "hash": "109ba9582f4bde945134e6794168af4c947a3cdf6d0391388cb09310fb811def"}
2024/10/30 07:25:45.015	DEBUG	tls.handshake	matched certificate in cache	{"remote_ip": "36.33.xx.xx", "remote_port": "49914", "subjects": ["36.33.xx.xx", "192.168.34.197"], "managed": false, "expiration": "2094/10/31 07:24:23.000", "hash": "109ba9582f4bde945134e6794168af4c947a3cdf6d0391388cb09310fb811def"}
```

### 其他的失败尝试

例如另起一个服务端口，然后使用反向代理提供服务：

```caddy
https://192.168.34.197:10443 {
        encode zstd gzip
        tls /home/caddy/herong/tls/3633xxxx-cert.pem /home/caddy/herong/tls/3633xxxx-key.pem
        reverse_proxy https://192.168.34.197 {
                header_up Host {upstream_hostport}
        }
}
https://192.168.34.197 {
	encode zstd gzip
	tls /home/caddy/herong/tls/19216834197-cert.pem /home/caddy/herong/tls/19216834197-key.pem
        handle {
                root * /home/caddy/herong/web/dist
                file_server
        }
}
```

我期望着第一个服务能够提供公网地址的证书，但是不行。

甚至用 HTTP 都不行：

```caddy
http://192.168.34.197:10443 {
        encode zstd gzip
        reverse_proxy https://192.168.34.197 {
                header_up Host {upstream_hostport}
        }
}
https://192.168.34.197 {
        encode zstd gzip
        tls /home/caddy/herong/tls/19216834197-cert.pem /home/caddy/herong/tls/19216834197-key.pem
        handle {
                root * /home/caddy/herong/web/dist
                file_server
        }
}
```

# 思考

1. 是不是提供 HTTPS 服务的设备必须同时拥有地址和证书？
2. 大型企业是否会遇到这样的问题？他们是怎么解决的？是不是有相关解决方案的供应商？
3. 如果在更低层的网络上进行编程，能否解决这个问题？例如在服务器的 TCP 层进行编程？是否已经有相关的解决方案？
4. 好像可以用 frp 解决？？！！

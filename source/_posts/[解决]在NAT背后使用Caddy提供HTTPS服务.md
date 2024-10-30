---
title: "[解决]在NAT背后使用Caddy提供HTTPS服务"
url: "[解决]在NAT背后使用Caddy提供HTTPS服务"
date: 2024-10-30 17:08:59
categories:
- 问题
tags:
- 问题
- 运维
- 项目实践
- HTTPS
- caddy
---

在 [问题](/\[问题\]在NAT背后使用Caddy提供HTTPS服务) 的最后，我突然想到可以用 frp 代理一个 HTTP 服务出去，这样虽然不能使用 https 的功能，但是起码可以访问其他的基础功能。

<!-- more -->

于是我做了这个：

```caddy
http://192.168.34.197:10443 {
        encode zstd gzip
        reverse_proxy https://192.168.34.197 {
                header_up Host {upstream_hostport}
                transport http {
                        tls_insecure_skip_verify
                }
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

发现只能内网访问，通过 frp 代理地址还是不能访问（只有 HTTP ok，没有内容）。

这一下打破了我原有的想法，我原本以为是网络和 TLS 问题，现在看来不是，然后注意到 `http://192.168.34.197:10443`，我开始怀疑是 caddy 的路由没匹配上。

改成 `:10443` 后，frp 可用了。

> caddy 中配置的 address 必须和用户在浏览器输入的地址相同。

后来把 Caddyfile 改成这样，直接按端口提供服务，一切问题都解决了：

```caddy
:443 {
        encode zstd gzip
        tls /home/caddy/herong/tls/19216834197-cert.pem /home/caddy/herong/tls/19216834197-key.pem
        handle {
                root * /home/caddy/herong/web/dist
                file_server
        }
}
```

也不需要按照浏览器的公网地址生成证书，反正都是信任不安全的证书，只要用内网地址生成一个证书就可以了。

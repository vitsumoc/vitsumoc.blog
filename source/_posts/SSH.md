---
title: SSH握手过程
url: SSH
date: 2023-11-20 09:20:06
categories:
- 网络编程
tags:
- 网络编程
- SSH
---

# RFC

https://datatracker.ietf.org/doc/html/rfc4253

# SSH简介

安全外壳协议（Secure Shell Protocol，简称SSH）是一种加密的网络传输协议，可在不安全的网络中为网络服务提供安全的传输环境。SSH通过在网络中建立安全隧道来实现SSH客户端与服务器之间的连接。SSH最常见的用途是远程登录系统，人们通常利用SSH来传输命令行界面和远程执行命令。

<!--more-->

# SSH数据包基本格式

SSH的数据包加密后分块传输，每次传输的实际包长度都应为密码块大小的整数倍或8

每个加密后的数据包都由如下结构构成

```c c
uint32    packet_length;
byte      padding_length;
byte[n1]  payload; // n1 = packet_length - padding_length - 1
byte[n2]  random_padding; // n2 = padding_length
byte[m]   mac(Message_Authentication_Code - MAC); // m = mac_length
```

+ packet_length：数据载荷的长度，不包括`mac`部分和`packet_length`本身。在进行加密协商完成后，传输的`packet_length`也会被加密

+ padding_length：`random_padding`块的大小

+ payload：数据载荷，根绝协商决定被加密或被压缩的方法

+ random padding： 0-255位随机填充

+ mac：信息认证码，用作信息完整性校验

# SSH过程

以下采用一个SSH抓包结果为例，描述SSH链接建立过程：

|=======|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|=======|

|&nbsp;&nbsp;&nbsp;客户端&nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;服务端&nbsp;&nbsp;&nbsp;&nbsp;|

|=======|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|=======|

|=======================链接建立=======================|

|1. 三次握手1|----------------------------------------------------------------------------------------->

<----------------------------------------------------------------------------------------|2. 三次握手2|

|3. 三次握手3|----------------------------------------------------------------------------------------->

|=======================协议协商=======================|

<----------------------------------------------------------------------------------------|4. 服务端协议|

|5. 客户端协议|----------------------------------------------------------------------------------------->

|=======================算法协商=======================|

<-------------------------------------------------------------------------------------|6. 服务端算法表|

|7. 客户端算法表|------------------------------------------------------------------------------------>

|=======================密钥交换=======================|

|8. Diffie-Hellman Init|------------------------------------------------------------------------------>

<------------------------------------------------|9. Diffie-Hellman Reply，New Keys，加密包|

|10. New Keys|---------------------------------------------------------------------------------------->

|=======================加密通讯=======================|

## 链接建立

（1）（2）（3）三次握手报文

[TCP三次握手](https://zh.wikipedia.org/zh-hans/%E4%BC%A0%E8%BE%93%E6%8E%A7%E5%88%B6%E5%8D%8F%E8%AE%AE)

## 协议协商

（4）服务端协议报文

Hex内容

> 0x 53 53 48 2d 32 2e 30 2d 4f 70 65 6e 53 53 48 5f 38 2e 30 0d 0a

报文内容

> SSH-2.0-OpenSSH_8.0&lt;CR>&lt;LF>

包括SSH、协议版本（2.0）、软件版本（OpenSSH_8.0）

（5）客户端协议报文

Hex内容

> 0x 53 53 48 2d 32 2e 30 2d 6e 73 73 73 68 32 5f 37 2e 30 2e 30 30 33 33 20 4e 65 74 53 61 72 61 6e 67 20 43 6f 6d 70 75 74 65 72 2c 20 49 6e 63 2e 0d 0a

报文内容

> SSH-2.0-nsssh2_7.0.0033 NetSarang Computer, Inc.&lt;CR>&lt;LF>

## 算法协商

在算法协商的过程中，双方会各自发送自己支持的算法列表，最终对以下几个算法达成共识：
+ kex_algorithms：密钥交换算法
+ server_host_key_algorithms：公钥算法
+ encryption_algorithms：加密算法
+ mac_algorithms：数据完整性算法
+ compression_algorithms：压缩算法
+ languages：语言标签（可选）
+ first_kex_packet_follows：表示是否有猜测数据包
  
在达成共识的过程中，基本以客户端中的算法排序优先匹配

（6）服务端算法表报文

+ packet_length：0x00 00 04 14（1044）
+ padding_length：0x05（5）
+ SSH_MSG_SERVICE_ACCEPT：0x14（`SSH_MSG_KEXINIT`）
+ Cookie：0xd7 86 29 66...(16Byte)
+ kex_algorithms length：下方算法表长度
+ kex_algorithms list：算法表（字符串表示，逗号分隔）
+ server_host_key_algorithms length：下方算法表长度
+ server_host_key_algorithms list：算法表（字符串表示，逗号分隔）
+ encryption_algorithms_client_to_server length：下方算法表长度
+ encryption_algorithms_client_to_server list：算法表（字符串表示，逗号分隔）
+ encryption_algorithms_server_to_client length：下方算法表长度
+ encryption_algorithms_server_to_client list：算法表（字符串表示，逗号分隔）
+ mac_algorithms_client_to_server length：下方算法表长度
+ mac_algorithms_client_to_server list：算法表（字符串表示，逗号分隔）
+ mac_algorithms_server_to_client length：下方算法表长度
+ mac_algorithms_server_to_client list：算法表（字符串表示，逗号分隔）
+ compression_algorithms_client_to_server length：下方算法表长度
+ compression_algorithms_client_to_server list：算法表（字符串表示，逗号分隔）
+ compression_algorithms_server_to_client length：下方算法表长度
+ compression_algorithms_server_to_client list：算法表（字符串表示，逗号分隔）
+ languages_client_to_server length：下方算法表长度
+ languages_client_to_server list：算法表（字符串表示，逗号分隔）
+ languages_server_to_client length：下方算法表长度
+ languages_server_to_client list：算法表（字符串表示，逗号分隔）
+ first_kex_packet_follows：0x00
+ Reserved：0x00 00 00 00
+ Padding：0x00 00 00 00 00（`padding_length`长度）

（7）客户端算法表报文

与服务端算法表格式相同

## 密钥交换

通过双方协商，决定采用Elliptic Curve Diffie-Hellman方式进行密钥交换

（8）客户端Diffie-Hellman Init
+ packet_length：0x00 00 00 2c
+ padding_length：0x06
+ MSG：0x1e（Elliptic Curve Diffie-Hellman Key Exchange Init）
+ 客户端公钥长度：0x00 00 00 20（32）
+ 客户端公钥：0xd1 d9 b8 6c 84 67 55 0f ca 84 6e 8b 0e 67 25 27 6b 50 ae ed a4 6d dc 0b 73 4c 15 ad e9 f5 51 66
+ Padding：0x91 f0 e8 0c f4 9b

（9）服务端Diffie-Hellman Reply，New Keys，加密包

服务端的回复包含三部分内容，Key Exchange Reply、New Keys、 加密包

其中，Key Exchange Reply包括了密钥交换的结果

+ packet_length：0x00 00 03 5c
+ padding_length：0x08
+ MSG：0x1f（Elliptic Curve Diffie-Hellman Key Exchange Reply）
+ Host Key Length：0x00 00 01 97
+ Host Key Type Length：0x00 00 00 07
+ Host Key Type：0x73 73 68 2d 72 73 21（ssh-rsa）
+ Multi Precision Integer Length：0x00 00 00 03
+ RSA public exponent (e)：0x01 00 01
+ Multi Precision Integer Length：0x00 00 01 81
+ RSA Modulus (N)：0x00 be 1b 4b 73 9d f8 37 0e 33...
+ ECDH server's ephemeral public key length：0x00 00 00 20
+ ECDH server's ephemeral public key (Q_S)：0x3a 2e 62 f6 ee...
+ KEX H signature length：0x00 00 01 8f
+ KEX H signature ：0x00 00 00 07 73 73 68 2d 72 73 61 00 00 01 80 a0...
+ Padding：0x00 00 00 00 00 00 00 00

New Keys表示密钥交换完成，此后的内容都需要使用新密钥处理

+ packet_length：0x00 00 00 0c
+ padding_length：0x0a
+ MSG：0x15（`SSH_MSG_NEWKEYS`）
+ Padding：0x00 00 00 00 00 00 00 00 00 00

后续的数据已经被加密，无法查看内容，推测是与客户端进行登录认证的协商

（10）客户端New Keys

客户端的New Keys包与服务端相同，后续客户端发送数据也都被加密处理
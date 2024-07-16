---
title: "[翻译]Sol - 从零开始的MQTT broker - 第一部分：协议"
url: translate-sol-1
date: 2023-12-08 15:30:15
categories:
- MQTT
tags:
- MQTT
- 翻译
- 网络编程
- 物联网
- C
---

> 原文 [Sol - An MQTT broker from scratch. Part 1 - The protocol](https://codepr.github.io/posts/sol-mqtt-broker/)

<!--more-->

# 前言

我已经在物联网领域工作有一段时间了，这段时间里我一直在处理物联网架构相关的工作，探索物联网系统开发的最佳模式，研究相关的协议和标准，例如MQTT。

因为我一直在渴望着提升我编程能力的机会，我觉得在物联网方向深入研究会很有趣也很有好处。因此，我再一次 `git init` 了一个项目，并且要通过写下这些博客来挑战我自己，强迫自己进步。

**Sol** 是一个C语言项目，一个超级简单的Linux平台的MQTT broker，支持MQTT 3.3.1，不兼容旧的版本，非常类似于轻量级的 `Mosquitto` （虽然这玩意已经是个轻量级软件了）。由于现在有很多种类的MQTT客户端，所以测试起来会比较简单。最终的成品可能会成为一个更简洁，功能更丰富的软件，我们要创造这个功能的最小化实现。顺便提一下，**Sol** 这个名字的来源有一半的原因是我对短名称的偏好，另一半的原因则是火星日 (The Martian docet)。或者说，**Sol** 可能代表**S**crappy **O**l' **L**oser。emmmm

**注意**：这个项目一直到最后才会编译，你需要跟写所有的代码步骤。如果你想要在中途进行测试，我建议你自己建一个主函数来做这些测试或者修改。

一步一步来，我一般会创建一个这样的文件结构来初始化我的C项目：

```text
sol/
 ├── src/
 ├── CHANGELOG
 ├── CMakeLists.txt
 ├── COPYING
 └── README.md
```

这里是Github上的[仓库](https://github.com/codepr/sol/tree/tutorial)。

我会尝试着一步一步描述 **Sol** 的开发过程，但我也不会贴上所有的代码，只会解释关键的地方。你想要学习的最好方式依然是亲自编写、编译、修改代码。

这将是一系列文章，每篇文章都将讨论并主要实施项目的一个概念/模块：

- [第一部分 ： 协议](https://codepr.github.io/posts/sol-mqtt-broker/) MQTT协议数据包处理的基础
- [第二部分 ： 网络](https://codepr.github.io/posts/sol-mqtt-broker-p2/) 解决网络通讯的功能模块
- [第三部分 ： 服务](https://codepr.github.io/posts/sol-mqtt-broker-p3/) 程序入口
- [第四部分 ： 数据结构](https://codepr.github.io/posts/sol-mqtt-broker-p4/) 常用数据结构实现
- [第五部分 ： 主题树](https://codepr.github.io/posts/sol-mqtt-broker-p5/) 通过特里树处理主题匹配
- [第六部分 ： 处理器](https://codepr.github.io/posts/sol-mqtt-broker-p6/) 每种数据包的处理函数
- [特别篇 ： 多线程](https://codepr.github.io/posts/sol-mqtt-broker-bonus/) 各种改进、bug修复、应用多线程

我想说，虽然 sol 会是一个完全功能的 broker，但仍有很大改进和优化空间，以及可能的一些隐藏功能（俗称BUG）。

# 架构设计

`broker` 的本质是一个中间件，它接受来自多个客户端（生产者）的输入，并使用抽象方法将其转发给一组目标客户端（消费者），这种抽象方法用于定义和管理这些客户端组，形式为 **channel** 或 **topic**（根据协议标准）。与 IRC 频道或通用聊天中的等效概念非常相似，每个消费者客户端都可以订阅 `topic`，以便接收其他客户端发布到这些 `topic` 的所有消息。

第一个想到的是建立在某种数据结构之上的服务器，这种数据结构可以轻松管理这些 `topic` 和连接的客户端（无论是生产者还是消费者）。客户端收到的每个消息都必须转发给所有订阅了该消息指定 `topic` 的其他已连接客户端。

让我们试试这种方法，使用一个 TCP 服务器和一个用于处理数据流的模块。实现服务器的方法有很多，包括线程、fork 进程和多路 I/O，这次我将尝试用多路 I/O 的方式。

我们先使用单线程多路 I/O 服务器，未来有可能进行多线程拓展。实际上，用于多路复用的 **epoll** 接口是线程安全的。

# MQTT结构

首先，我们需要基于[官方文档](http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/errata01/os/mqtt-v3.1.1-errata01-os-complete.html)，制作一些描述 MQTT 协议数据包的结构体。

从 opcode 表和 MQTT 头开始，基于文档，每个数据包都包含以下三部分：

- fixed header（必选）
- variable header（可选）
- payload（可选）

## Fixed Header

Fixed Header的第一个字节包括了 `MQTT type` 和 `Flags`，第二到第五个字节使用可变编码的方式，存储剩余数据包的长度。

```text Fixed Header

 | Bit    | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
 |--------|---------------|---------------|
 | Byte 1 | MQTT type     |  Flags        |
 |--------|-------------------------------|
 | Byte 2 |                               |
 |  .     |      Remaining Length         |
 |  .     |                               |
 | Byte 5 |                               |
```

Flags并不是强制填写的，只是一些控制类数据，内容如下：

- Dup flag： 当消息被发送超过一次时使用
- QoS level： 有以下三种取值 `AT_MOST_ONCE`=0， `AT_LEAST_ONCE`=1 and `EXACTLY_ONCE`=2
- Retain flag： 保留标志，有保留标志的消息被发布到主题时，消息会被保留，之后连接进来的客户端也可以收到该消息。保留消息可以被另一条保留消息覆盖。

所以，打开 Vim （或者其他任何你喜欢的IDE），创建名为 `mqtt.h` 的头文件，开始写关于 Fixed Header 的数据结构吧：

```c src/mqtt.h
#include <stdio.h>

#define MQTT_HEADER_LEN 2
#define MQTT_ACK_LEN    4

/*
 * 回复信息枚举，用于 Fixed Header 中的第一个字节
 * 准确的说是只负责设置高位的4bit
 */
#define CONNACK_BYTE  0x20
#define PUBLISH_BYTE  0x30
#define PUBACK_BYTE   0x40
#define PUBREC_BYTE   0x50
#define PUBREL_BYTE   0x60
#define PUBCOMP_BYTE  0x70
#define SUBACK_BYTE   0x90
#define UNSUBACK_BYTE 0xB0
#define PINGRESP_BYTE 0xD0

/* 信息类型 */
enum packet_type {
    CONNECT     = 1,
    CONNACK     = 2,
    PUBLISH     = 3,
    PUBACK      = 4,
    PUBREC      = 5,
    PUBREL      = 6,
    PUBCOMP     = 7,
    SUBSCRIBE   = 8,
    SUBACK      = 9,
    UNSUBSCRIBE = 10,
    UNSUBACK    = 11,
    PINGREQ     = 12,
    PINGRESP    = 13,
    DISCONNECT  = 14
};

enum qos_level { AT_MOST_ONCE, AT_LEAST_ONCE, EXACTLY_ONCE };

union mqtt_header {
    unsigned char byte;         // 将 header 视为一个byte操作
    struct {                    // 将 header 视为内部结构分开操作
        unsigned retain : 1;    // 保留标识
        unsigned qos : 2;       // qos标识
        unsigned dup : 1;       // 重复标识
        unsigned type : 4;      // 4bit Flags
    } bits;
};
```

最上方的两个 `#define` 定义了 MQTT Fixed Header 和 MQTT ACK 的长度。

正如你在代码中看到的，我们利用了 **union**——一种可以在内存中的同一位置存储多种表示形式的结构——来表示一个字节。换句话说，与普通的 `struct` 不同，`union` 中只能有一个字段具有值（在此例中是byte或bits）。它们的内存位置是共享的，因此，通过使用**位字段**，我们可以有效地操作单个比特或字节的一部分。

## CONNECT

我们要定义的第一个控制数据包是 CONNECT。 这是当客户端建立新连接时必须发送的第一个数据包，CONNECT 包必须是有且仅有一个，否则视为与协议不符，服务端需要断开连接。

对于每个 CONNECT，服务端需要在响应中回复 CONNACK。

```c src/mqtt.h
struct mqtt_connect {
    union mqtt_header header;               // 第一个byte是通用头
    union {                                 // 第二个byte表示一些控制信息
        unsigned char byte;
        struct {
            int reserved : 1;
            unsigned clean_session : 1;     // 为1时表示新session，否则表示已有session
            unsigned will : 1;              // 表示是否有遗嘱
            unsigned will_qos : 2;          // 表示遗嘱的QOS
            unsigned will_retain : 1;       // 表示遗嘱发布时是否保留
            unsigned password : 1;          // 表示是否有密码
            unsigned username : 1;          // 表示是否有用户名
        } bits;
    };
    struct {                                // 载荷
        unsigned short keepalive;           // 会话保活时间，单位秒
        unsigned char *client_id;
        unsigned char *username;
        unsigned char *password;
        unsigned char *will_topic;
        unsigned char *will_message;
    } payload;
};

struct mqtt_connack {
    union mqtt_header header;
    union {
        unsigned char byte;
        struct {
            unsigned session_present : 1;
            unsigned reserved : 7;
        } bits;
    };
    unsigned char rc; // return code 返回值
};
```

按照这个模式，结合 MQTT v3.1.1 的文档，其他数据包的定义也比较简单了。

## SUBSCRIBE UNSUBSCRIBE PUBLISH ACK等

接下来我们处理 SUBSCRIBE，UNSUBSCRIBE 和 PUBLISH。SUBSCRIBE 必须要使用 SUBACK 来响应，其他的都可以使用通用 ACK，并设置 **typedef** 字段的值来响应。

```c src/mqtt.h
struct mqtt_subscribe {
    union mqtt_header header;
    unsigned short pkt_id;
    unsigned short tuples_len;      // 接下来数据元组的长度
    struct {
        unsigned short topic_len;   // 接下来 topic 字符串的长度
        unsigned char *topic;
        unsigned qos;
    } *tuples;
};

struct mqtt_unsubscribe {
    union mqtt_header header;
    unsigned short pkt_id;
    unsigned short tuples_len;
    struct {
        unsigned short topic_len;
        unsigned char *topic;
    } *tuples;
};

struct mqtt_suback {                // 针对 SUB 动作的响应
    union mqtt_header header;
    unsigned short pkt_id;
    unsigned short rcslen;
    unsigned char *rcs;
};

struct mqtt_publish {               // 发布消息
    union mqtt_header header;
    unsigned short pkt_id;
    unsigned short topiclen;
    unsigned char *topic;
    unsigned short payloadlen;
    unsigned char *payload;
};

struct mqtt_ack {                   // 通用响应
    union mqtt_header header;
    unsigned short pkt_id;
};
```

剩余的这一类ACK包：

- PUBACK
- PUBREC
- PUBREL
- PUBCOMP
- UNSUBACK
- PINGREQ
- PINGRESP
- DISCONNECT

因为有相同的结构，所以都可以通过 typedef 来定义，只是语义有所不同。最后一个 DISCONNECT，虽然严格来说不是一个 ACK，但是也有相同的结构。

```c src/mqtt.h
typedef struct mqtt_ack mqtt_puback;
typedef struct mqtt_ack mqtt_pubrec;
typedef struct mqtt_ack mqtt_pubrel;
typedef struct mqtt_ack mqtt_pubcomp;
typedef struct mqtt_ack mqtt_unsuback;
typedef union mqtt_header mqtt_pingreq;
typedef union mqtt_header mqtt_pingresp;
typedef union mqtt_header mqtt_disconnect;
```

## MQTT

最终我们可以定义一个通用 MQTT 包，包括上面的一切，后续我们所有的 MQTT 数据包都可以用这个结构来表示。

```c src/mqtt.h
union mqtt_packet {
    struct mqtt_ack ack;                    // 通用ACK
    union mqtt_header header;               // 通用头
    struct mqtt_connect connect;            // CONNECT包 (这种包里会包括一个通用头)
    struct mqtt_connack connack;            // CONNACK包
    struct mqtt_suback suback;              // SUBBACK包
    struct mqtt_publish publish;            // PUBLISH包
    struct mqtt_subscribe subscribe;        // SUB包
    struct mqtt_unsubscribe unsubscribe;    // UNSUB包
};
```

# MQTT函数

## 编码解码

现在我们继续定义一些公共函数。在 `src/mqtt.h` 中，我们需要考虑其他模块使用 MQTT 协议时会用到哪些函数。

为了使用 MQTT 协议处理通信，我们基本上需要 4 个函数，其中客户端向服务端有 2 个，服务端向客户端也是 2 个：

- 一个编码函数（总之就是把内存里的数据做成二进制流，这里不讨论术语）
- 一个解码函数（就是从二进制流恢复成内存结构）

我们还需要 2 个函数来处理 fixed head 部分中变长的 Remaining Length 字段。

```c src/mqtt.h
// 编码时生成 Remaining Length
int mqtt_encode_length(unsigned char *, size_t);    // size_t 指uint32 或 uint64
// 解码时解析 Remaining Length
unsigned long long mqtt_decode_length(const unsigned char **);
// 将char * 解码为 mqtt_packet *
int unpack_mqtt_packet(const unsigned char *, union mqtt_packet *);
// 将 mqtt_packet * 编码为 char *
unsigned char *pack_mqtt_packet(const union mqtt_packet *, unsigned);   // unsigned指 unsigned int
```

## 内存操作

我们还需要一些工具函数，用来进行基于数据包的内存分配、释放，这里没啥特别的。

```c src/mqtt.h
// 申请内存，制作各种MQTT包
union mqtt_header *mqtt_packet_header(unsigned char);
struct mqtt_ack *mqtt_packet_ack(unsigned char , unsigned short);
struct mqtt_connack *mqtt_packet_connack(unsigned char ,
                                         unsigned char ,
                                         unsigned char);
struct mqtt_suback *mqtt_packet_suback(unsigned char, unsigned short,
                                       unsigned char *, unsigned short);
struct mqtt_publish *mqtt_packet_publish(unsigned char, unsigned short, size_t,
                                         unsigned char *,
                                         size_t, unsigned char *);
// 释放MQTT包
void mqtt_packet_release(union mqtt_packet *, unsigned);
```

# 函数实现

## MQTT包编解码接口

好了，我们现在有一个不错的头文件了，定义了我们通讯协议中的所有内容，现在我们需要实现这些函数了。为了能够实现这些功能，首先我们要定义几个**私有**的帮助函数，用来进行编码和解码的动作。这些函数会被**公有**函数`unpack_mqtt_packet` 和 `pack_mqtt_packet` 调用。

```c src/mqtt.c
#include <stdlib.h>
#include <string.h>
#include "mqtt.h"

// 一系列对于具体类型包的 pack unpack 函数
static size_t unpack_mqtt_connect(const unsigned char *,
                                  union mqtt_header *,
                                  union mqtt_packet *);
static size_t unpack_mqtt_publish(const unsigned char *,
                                  union mqtt_header *,
                                  union mqtt_packet *);
static size_t unpack_mqtt_subscribe(const unsigned char *,
                                    union mqtt_header *,
                                    union mqtt_packet *);
static size_t unpack_mqtt_unsubscribe(const unsigned char *,
                                      union mqtt_header *,
                                      union mqtt_packet *);
static size_t unpack_mqtt_ack(const unsigned char *,
                              union mqtt_header *,
                              union mqtt_packet *);
static unsigned char *pack_mqtt_header(const union mqtt_header *);
static unsigned char *pack_mqtt_ack(const union mqtt_packet *);
static unsigned char *pack_mqtt_connack(const union mqtt_packet *);
static unsigned char *pack_mqtt_suback(const union mqtt_packet *);
static unsigned char *pack_mqtt_publish(const union mqtt_packet *);
```

## 二进制流编解码实现

在继续实现 `src/mqtt.h` 上所有定义的函数之前，我们需要实现一些辅助函数，以简化每个接收到的数据包的编码解码过程。

让我们快速搞定这部分，这一块只是简单的序列化和反序列化操作而已（记得用Big-endian就行）。

```c src/pack.h
#include <stdio.h>
#include <stdint.h>

/* 从数据流中获得数据的方法 */
// bytes -> uint8_t
uint8_t unpack_u8(const uint8_t **);
// bytes -> uint16_t
uint16_t unpack_u16(const uint8_t **);
// bytes -> uint32_t
uint32_t unpack_u32(const uint8_t **);
// 读取定义的 len 个字节（用来读取字符串）
uint8_t *unpack_bytes(const uint8_t **, size_t, uint8_t *);
// 读取字符串前面的 ushort 长度，并申请 dest内存块存字符串
uint16_t unpack_string16(uint8_t **buf, uint8_t **dest)
/* 将数据写入数据流的方法 */
// append a uint8_t -> bytes into the bytestring
void pack_u8(uint8_t **, uint8_t);
// append a uint16_t -> bytes into the bytestring
void pack_u16(uint8_t **, uint16_t);
// append a uint32_t -> bytes into the bytestring
void pack_u32(uint8_t **, uint32_t);
// 将 len 个字节追加到bytes中
void pack_bytes(uint8_t **, uint8_t *);
```

以及相应的实现

```c src/pack.c
#include <string.h>
#include <stdlib.h>
#include <arpa/inet.h>
#include "pack.h"

// Reading data
uint8_t unpack_u8(const uint8_t **buf) {
    uint8_t val = **buf;
    (*buf)++;
    return val;
}

uint16_t unpack_u16(const uint8_t **buf) {
    uint16_t val;
    memcpy(&val, *buf, sizeof(uint16_t));
    (*buf) += sizeof(uint16_t);
    return ntohs(val);
}

uint32_t unpack_u32(const uint8_t **buf) {
    uint32_t val;
    memcpy(&val, *buf, sizeof(uint32_t));
    (*buf) += sizeof(uint32_t);
    return ntohl(val);
}

uint8_t *unpack_bytes(const uint8_t **buf, size_t len, uint8_t *str) {
    memcpy(str, *buf, len);
    str[len] = '\0';
    (*buf) += len;
    return str;
}

uint16_t unpack_string16(uint8_t **buf, uint8_t **dest) {
    uint16_t len = unpack_u16(buf);
    *dest = malloc(len + 1);
    *dest = unpack_bytes(buf, len, *dest);
    return len;
}

// Write data
void pack_u8(uint8_t **buf, uint8_t val) {
    **buf = val;
    (*buf) += sizeof(uint8_t);
}

void pack_u16(uint8_t **buf, uint16_t val) {
    uint16_t htonsval = htons(val);
    memcpy(*buf, &htonsval, sizeof(uint16_t));
    (*buf) += sizeof(uint16_t);
}

void pack_u32(uint8_t **buf, uint32_t val) {
    uint32_t htonlval = htonl(val);
    memcpy(*buf, &htonlval, sizeof(uint32_t));
    (*buf) += sizeof(uint32_t);
}

void pack_bytes(uint8_t **buf, uint8_t *str) {
    size_t len = strlen((char *) str);
    memcpy(*buf, str, len);
    (*buf) += len;
}
```

这样我们就完成了字节流和数据类型的双向转换工作。

## Remaining Length编解码实现

完成了 `pack` 部分后，我们需要把他们运用在我们的MQTT包里，首先当然是：

```c src/mqtt.c
#include "pack.h"
```

第一步我们可以实现对 Fixed Header 中的 Remaining Length 字段的操作。MQTT文档中提供了这一段实现的伪代码，我们可以仿写一下。

让我们来看看 Remaining Length 如何用1-4个变长的Byte来表示剩余包的长度。

> Remaining Length 表示的是数据包剩余部分的长度，包括 variable header 和 payload。Remaining Length 中表示的长度不包括 Remaining Length 字段本身所占用的长度。
>
> Remaining Length 的编码使用了一种可变长度编码方案，该方案对 127 以下的值使用单个字节。较大的值则按以下方式处理：每个字节的低 7 位编码数据，高位用于指示是否存在后续字节。因此，每个字节编码 128 个值和一个 "延续位"。Remaining Length 字段的最大字节数为 4。

MQTT的文档已经描述的非常清晰，我们只需要实现。

```c src/mqtt.c
/*
 * 基于 MQTT v3.1.1，Fixed Header 中的 Remaining Length 最大为4byte
 */
static const int MAX_LEN_BYTES = 4;

/*
 * 根据数据包长度制作变长的 Remaining Length
 * return Remaining Length 的字节长度
 * buf Remaining Length 的数据流
 * len Remaining Length 应该表示的值（可变头+载荷总长度）
 */
int mqtt_encode_length(unsigned char *buf, size_t len) {
    // 字节长度
    int bytes = 0;
    do {
        if (bytes + 1 > MAX_LEN_BYTES)
            return bytes;
        short d = len % 128;
        len /= 128;
        // len > 0 表示还有后续位
        if (len > 0)
            d |= 128; // 标记最高位
        buf[bytes++] = d;
    } while (len > 0);
    return bytes;
}

/*
 * 解析数据流中的 Remaing Length 并将指针移动到下一个位置
 * return Remaining Length 的值
 * buf Remaining Length 的数据流
 *
 * TODO Handle case where multiplier > 128 * 128 * 128
 */
unsigned long long mqtt_decode_length(const unsigned char **buf) {
    char c;
    // 乘数
    int multiplier = 1;
    // 值
    unsigned long long value = 0LL;
    do {
        c = **buf;
        value += (c & 127) * multiplier;
        multiplier *= 128;
        // 后移一位
        (*buf)++;
        // 当没有后续位标识时结束
    } while ((c & 128) != 0);
    return value;
}
```

## CONNECT 解码实现

好了，现在我们可以完整的解析 Fixed Header 了，接下来我们试着解码 CONNECT 包。

CONNECT 是一个有很多flags的包，而且长度仅次于 PUBLISH 包。

CONNECT 包的内容包括：

- Fixed Header 中的 MQTT type + Flags，高4位（MQTT type）（称为**MSB**）的值是`1`，表示`Connect type`，低4位（Flags）（**LSB**）保留
- Fixed Header 中的变长 Remaining Length，表示剩余部分的长度
- Variable Header，由四个字段组成：
    - Protocol Name
    - Protocol Level
    - Connect Flags
    - Keep Alive
- 可能存在或者不存在的 payload（基于 Connect Flags 的设置）

> Protocol Name 是 UTF-8 编码的大写字符串 "MQTT"，这个字段的长度和内容在未来版本的MQTT协议中都不会再改变。

所以 3.1.1 版本的 Protocol Name 就是 "MQTT"，我们也不用去管旧版本的名字是什么了。

Connect flags 为一个byte，包含了一些关于客户端行为以及是否有 payload 段存在的标识：

|  Connect flags 中的字段   | 大小  | 含义 |
| :----: | :----: | :----: |
| Username flag | 1bit | 表示用户名存在与否 |
| Password flag | 1bit | 表示密码存在与否 |
| Will retain | 1bit | 表示遗嘱是否保留 |
| Will QoS | 2bit | 表示遗嘱的QOS等级 |
| Will flag | 1bit | 表示遗嘱存在与否 |
| Clean Session | 1bit | 表示是否为新链接 |

Connect flags的最高位保留，其他所有位都被当作bool值初始化（除了Will QoS），这些bool值在 payload 部分也有相应的字段。比如当 Username 和 Password 的值为1，那么在 payload 中会有 2byte 的 username length，紧随其后的就是 username 字符串，Password也是相同的道理。

为了说明这件事，假设我们收到了这样一个 CONNECT 包：

- Connect flags 中的 username 和 password 都置为1
- username = "hello"
- password = "nacho"
- client ID = "danzan"

那么这个数据包应该长这样：

| 字段 | 大小 | 偏移量 | 描述 |
| ---- | :----: | :----: | ---- |
| Packet type + Falgs | 1 | 0 | 类型为`Connect type` `0x01`，Flags为空 |
| Length | 1 | 1 | 后续总长度32Byte，小于127，所以可以用1Byte表示 |
| Protocol name length | 2 | 2 | 协议名长度，值固定为 `0x04` |
| Protocol name | 4 | 4 | 'M' 'Q' 'T' 'T' |
| Protocol level | 1 | 8 | 对于MQTT 3.1.1 此字段值为 `0x04` |
| Connect flags | 1 | 9 | 包括 `Username`, `password`, `will retain`, `will QoS`, `will flag`, `clean session` |
| Keepalive | 2 | 10 | ushort，保活时间，单位秒，最大值65536（18小时12分15秒） |
| Client ID length | 2 | 12 | ushort, 此例中值为`0x06` (danzan) |
| Client ID | 6 | 14 | 'd' 'a' 'n' 'z' 'a' 'n' |
| Username length | 2 | 20 | ushort, 此例中值为`0x05` (hello) |
| Username | 5 | 22 | 'h' 'e' 'l' 'l' 'o' |
| Password length | 2 | 27 | ushort,  此例中值为`0x05` (nacho) |
| Password | 5 | 29 | 'n' 'a' 'c' 'h' 'o' |

例如因为 Will Flags 被置为0，所以我们不需要在 `payload` 中解析这个字段（也压根没有），上例中我们要解析的内容总共就是包括 Fixed Header 在内的 34个byte。

```c src/mqtt.c
/*
 * CONNECT 解码函数
 * return Remaing Length 的值
 * buf 数据流，从变长长度开始
 * hdr 已经解码好的头部
 * pkt 返回的解码后数据包
 */
static size_t unpack_mqtt_connect(const unsigned char *buf,
                                  union mqtt_header *hdr,
                                  union mqtt_packet *pkt) {
    // 制作一个connect结构体，并且用已经解码好的头部赋值
    // 此处有一个已经解码好的头部，是因为数据作为二进制流进来的时候，肯定是要先解码出头部，然后再根据包类型分到不同的函数里做进一步解码的
    struct mqtt_connect connect = { .header = *hdr };
    // 将这个结构体赋值到pkt
    pkt->connect = connect;
    // 初始指针指向buf的首部
    const unsigned char *init = buf;
    /*
     * 获得后续的变长总长度,同时将指针移动到 protocol name
     */
    size_t len = mqtt_decode_length(&buf);
    // 暂时忽略协议名称、保留字段等等，所以直接向后移动8byte
    // 这里 init 直接+8，暗示了变长长度字段的长度是1byte，所以才能+8后指向Connect flags
    buf = init + 8;
    // 读取 Connect flags
    pkt->connect.byte = unpack_u8((const uint8_t **) &buf);
    // 读取 keepalive
    pkt->connect.payload.keepalive = unpack_u16((const uint8_t **) &buf);
    // 读取 CID 长度（如果有CID则>0，否则为0）
    uint16_t cid_len = unpack_u16((const uint8_t **) &buf);
    // 如果有，则读取CID
    if (cid_len > 0) {
        pkt->connect.payload.client_id = malloc(cid_len + 1);
        unpack_bytes((const uint8_t **) &buf, cid_len,
                     pkt->connect.payload.client_id);
    }
    // 如果有，则读取遗嘱
    if (pkt->connect.bits.will == 1) {
        unpack_string16(&buf, &pkt->connect.payload.will_topic);
        unpack_string16(&buf, &pkt->connect.payload.will_message);
    }
    // 如果有，则读取用户名
    if (pkt->connect.bits.username == 1)
        unpack_string16(&buf, &pkt->connect.payload.username);
    // 如果有，则读取密码
    if (pkt->connect.bits.password == 1)
        unpack_string16(&buf, &pkt->connect.payload.password);
    return len;
}
```

<!-- read -->

## PUBLISH 解码实现

以下是 PUBLISH 包的结构：

```
 |   Bit    |  7  |  6  |  5  |  4  |  3  |  2  |  1  |   0    |  <-- Fixed Header
 |----------|-----------------------|--------------------------|
 | Byte 1   |      MQTT type 3      | dup |    QoS    | retain |
 |----------|--------------------------------------------------|
 | Byte 2   |                                                  |
 |  .       |               Remaining Length                   |
 |  .       |                                                  |
 | Byte 5   |                                                  |
 |----------|--------------------------------------------------|  <-- Variable Header
 | Byte 6   |                Topic len MSB                     |
 | Byte 7   |                Topic len LSB                     |
 |-------------------------------------------------------------|
 | Byte 8   |                                                  |
 |   .      |                Topic name                        |
 | Byte N   |                                                  |
 |----------|--------------------------------------------------|
 | Byte N+1 |            Packet Identifier MSB                 |
 | Byte N+2 |            Packet Identifier LSB                 |
 |----------|--------------------------------------------------|  <-- Payload
 | Byte N+3 |                   Payload                        |
 | Byte N+M |                                                  |
```

仅当 QoS level > 0 时，存在 Packet identifier MSB 和 LSB。当 QoS 被设置为 *at most once* （值为0）时，没有必要存在 packet ID。

Payload部分的长度通过 Remaining Length 减去其他所有内容计算得来。

```c src/mqtt.c
/*
 * PUBLISH 解码函数
 * return Remaing Length 的值
 * buf 数据流，从变长长度开始
 * hdr 已经解码好的头部
 * pkt 返回的解码后数据包
 */
static size_t unpack_mqtt_publish(const unsigned char *buf,
                                  union mqtt_header *hdr,
                                  union mqtt_packet *pkt) {
    // 创建 PUBLISH 包并且使用已经解码好的 header 赋值
    struct mqtt_publish publish = { .header = *hdr };
    // 准备给返回值提供这个 PUBLISH 包
    pkt->publish = publish;
    // 通过变长的 Remaing Length 字段获取剩余部分的长度
    size_t len = mqtt_decode_length(&buf);
    // 获得 topiclen 和 topic 内容
    pkt->publish.topiclen = unpack_string16(&buf, &pkt->publish.topic);
    // 将 len 赋值, 并视为 payload 长度
    uint16_t message_len = len;
    // 如果 QoS > 0, 需要读取pkt_id
    if (publish.header.bits.qos > AT_MOST_ONCE) {
        pkt->publish.pkt_id = unpack_u16((const uint8_t **) &buf);
        // 此时payload长度需要减去pkt_id
        message_len -= sizeof(uint16_t);
    }
    // payload 长度需要减去 topic_len 字段长度和 topic 字段实际长度
    message_len -= (sizeof(uint16_t) + topic_len);
    // 这里是正确的 payloadlen
    pkt->publish.payloadlen = message_len;
    // 读取 payload
    pkt->publish.payload = malloc(message_len + 1);
    unpack_bytes((const uint8_t **) &buf, message_len, pkt->publish.payload);
    return len;
}
```

## SUBSCRIBE 和 UNSUBSCRIBE 解码实现

SUBSCRIBE 包和 UNSUBSCRIBE 包的结构非常相似。他们的 payload 部分都是一个 topic 相关的元组列表，其中 SUBSCRIBE 的元组是 (topic_len, topic_filter, qos)，而 UNSUBSCRIBE 是 (topic_len, topic_filter)。

```c src/mqtt.c
/*
 * SUBSCRIBE 解码函数
 * return Remaing Length 的值
 * buf 数据流，从变长长度开始
 * hdr 已经解码好的头部
 * pkt 返回的解码后数据包
 */
static size_t unpack_mqtt_subscribe(const unsigned char *buf,
                                    union mqtt_header *hdr,
                                    union mqtt_packet *pkt) {
    // 创建 SUBSCRIBE 包并且使用已经解码好的 header 赋值
    struct mqtt_subscribe subscribe = { .header = *hdr };
    // 通过变长的 Remaing Length 字段获取剩余部分的长度
    size_t len = mqtt_decode_length(&buf);
    size_t remaining_bytes = len;
    // 读取pkt_id
    subscribe.pkt_id = unpack_u16((const uint8_t **) &buf);
    remaining_bytes -= sizeof(uint16_t);
    /*
     * 订阅频道列表, 由一系列三元组构成
     *  - topic length 主题字符串长度
     *  - topic filter (string) 主题filter
     *  - qos
     */
    int i = 0;
    while (remaining_bytes > 0) {
        // 减去2byte, 是topic length的空间
        remaining_bytes -= sizeof(uint16_t);
        // 给这个主题字符串分配内存
        subscribe.tuples = realloc(subscribe.tuples,
                                   (i+1) * sizeof(*subscribe.tuples));
        // 获得主题字符串长度, 获得主题字符串内容
        subscribe.tuples[i].topic_len =
            unpack_string16(&buf, &subscribe.tuples[i].topic);
        // 减去主题字符串实际占用的空间
        remaining_bytes -= subscribe.tuples[i].topic_len;
        // 获得主题qos
        subscribe.tuples[i].qos = unpack_u8((const uint8_t **) &buf);
        // 减去主题 qos 的空间
        len -= sizeof(uint8_t);
        // 操作下一个主题
        i++;
    }
    // 记录订阅主题数
    subscribe.tuples_len = i;
    // 记录到 mqtt_packet
    pkt->subscribe = subscribe;
    return len;
}

/*
 * UNSUBSCRIBE 解码函数
 * return Remaing Length 的值
 * buf 数据流，从变长长度开始
 * hdr 已经解码好的头部
 * pkt 返回的解码后数据包
 */
static size_t unpack_mqtt_unsubscribe(const unsigned char *buf,
                                      union mqtt_header *hdr,
                                      union mqtt_packet *pkt) {
    struct mqtt_unsubscribe unsubscribe = { .header = *hdr };
    /*
     * Second byte of the fixed header, contains the length of remaining bytes
     * of the connect packet
     */
    size_t len = mqtt_decode_length(&buf);
    size_t remaining_bytes = len;
    /* Read packet id */
    unsubscribe.pkt_id = unpack_u16((const uint8_t **) &buf);
    remaining_bytes -= sizeof(uint16_t);
    /*
     * Read in a loop all remaining bytes specified by len of the Fixed Header.
     * From now on the payload consists of 2-tuples formed by:
     *  - topic length
     *  - topic filter (string)
     */
    int i = 0;
    while (remaining_bytes > 0) {
        /* Read length bytes of the first topic filter */
        remaining_bytes -= sizeof(uint16_t);
        /* We have to make room for additional incoming tuples */
        unsubscribe.tuples = realloc(unsubscribe.tuples,
                                     (i+1) * sizeof(*unsubscribe.tuples));
        unsubscribe.tuples[i].topic_len =
            unpack_string16(&buf, &unsubscribe.tuples[i].topic);
        remaining_bytes -= unsubscribe.tuples[i].topic_len;
        i++;
    }
    unsubscribe.tuples_len = i;
    pkt->unsubscribe = unsubscribe;
    return len;
}
```

## ACK 解码实现

最终到了 ACK 包，MQTT 协议中没有设计通用 ACK，但是实际上每个 ACK 包的数据结构都是一样的，有一个 Fixed Header 和一个 packet_id组成。

MQTT 协议中有如下几种类型的ACK:

- PUBACK
- PUBREC
- PUBREL
- PUBCOMP
- UNSUBACK

```c src/mqtt.c
/*
 * ACK 解码函数
 * return Remaing Length 的值
 * buf 数据流，从变长长度开始
 * hdr 已经解码好的头部
 * pkt 返回的解码后数据包
 */
static size_t unpack_mqtt_ack(const unsigned char *buf,
                              union mqtt_header *hdr,
                              union mqtt_packet *pkt) {
    // 创建 ACK 包并且使用已经解码好的 header 赋值
    struct mqtt_ack ack = { .header = *hdr };
    // 通过变长的 Remaing Length 字段获取剩余部分的长度
    size_t len = mqtt_decode_length(&buf);
    // pkt_id
    ack.pkt_id = unpack_u16((const uint8_t **) &buf);
    pkt->ack = ack;
    return len;
}
```

## MQTT包解码实现

现在我们已经实现了 `unpack_mqtt_packet` 需要的所有工具函数，接下来我们先定义一个解码函数的接口，然后使用一个静态数组来索引所有的解码函数，这里我们直接使用 `Control Packet type` 的值来作为数组中的索引。

需要注意的是，`DISCONNECT` `PINGREQ` `PINGRESP` 这三种包只有一个byte，所以我们不需要编写解码工具函数。

```c src/mqtt.c
// 解码函数接口
typedef size_t mqtt_unpack_handler(const unsigned char *,
                                   union mqtt_header *,
                                   union mqtt_packet *);

// 所有解码函数的列表, 索引值和包类型对应
static mqtt_unpack_handler *unpack_handlers[11] = {
    NULL,
    unpack_mqtt_connect,
    NULL,
    unpack_mqtt_publish,
    unpack_mqtt_ack,
    unpack_mqtt_ack,
    unpack_mqtt_ack,
    unpack_mqtt_ack,
    unpack_mqtt_subscribe,
    NULL,
    unpack_mqtt_unsubscribe
};

// MQTT 包解码入口
int unpack_mqtt_packet(const unsigned char *buf, union mqtt_packet *pkt) {
    int rc = 0;
    // 第一个 byte 是 fiexd header 中的 mqttType + flags
    unsigned char type = *buf;
    // 第一个byte可以被作为header
    union mqtt_header header = {
        .byte = type
    };
    // 对于这些包暂时无需解码
    if (header.bits.type == DISCONNECT
        || header.bits.type == PINGREQ
        || header.bits.type == PINGRESP)
        pkt->header = header;
    else
        // 通过包类型找到解码函数, 执行解码操作后返回rc, 此时rc等于具体解码函数的返回值
        rc = unpack_handlers[header.bits.type](++buf, &header, pkt);
    return rc;
}
```

# 结尾

从零开始MQTT broker的第一部分就这样结束了，我们做了两个模块，一个根据 OASIS 定义的标准描述MQTT协议结构，另一个则用来处理编解码操作。

此时我们的文件结构是这样的：

```text
sol/
 ├── src/
 │    ├── mqtt.h
 │    ├── mqtt.c
 │    ├── pack.h
 │    └── pack.c
 ├── CHANGELOG
 ├── CMakeLists.txt
 ├── COPYING
 └── README.md
```
---
title: "[翻译]Sol - 从零开始的MQTT broker - 第二部分：网络"
url: translate-sol-2
date: 2023-12-19 17:13:46
categories:
- MQTT
tags:
- MQTT
- 翻译
- 网络编程
- 物联网
- C
---

> 原文 [Sol - An MQTT broker from scratch. Part 2 - Networking](https://codepr.github.io/posts/sol-mqtt-broker-p2/)

<!--more-->

# 前言

让我们继续之前的工作，在第一部分中我们实现了 MQTT v3.1.1 的数据结构和解码函数，接下来我们需要做一些组包和编码函数，让我们可以发送网络包。

顺带说明一下，我们并没有打算去编写完美的或者内存效率很高的代码，而且，过早的优化是万恶之源，以后我们有的是时间来提高我们的代码质量。

# 组包实现

暂时我们只需要做 `CONNACK` `SUBACK` `PUBLISH` 包的组包工作，其他的各种 `ACK` 的结构都是一样的，之前我们已经用 **typedef** 让这些 `ACK` 引用了同一个函数。

- `union mqtt_header *mqtt_packet_header(unsigned char)` 函数用来处理 Fixed Header，以及以下这些只有 Fixed Header 的包：
  - PINGREQ
  - PINGRESP
  - DISCONNECT

- `struct mqtt_ack *mqtt_packet_ack(unsigned char, unsigned short)` 用来处理以下这些 `类ACK` 的包：
  - PUBACK
  - PUBREC
  - PUBREL
  - PUBCOMP
  - UNSUBACK

其余的包都需要专门的函数来组包。再说一次，虽然可能有很多更优雅的代码或者更优化的方法，但是现在我们只要写能用的代码就行了，以后迟早会优化的。

```c mqtt.c
/*
 * mqtt组包
 */

// 头部1byte的组包实现
union mqtt_header *mqtt_packet_header(unsigned char byte) {
    static union mqtt_header header;
    header.byte = byte;
    return &header;
}

// 各种ACK的组包实现
struct mqtt_ack *mqtt_packet_ack(unsigned char byte, unsigned short pkt_id) {
    static struct mqtt_ack ack;
    ack.header.byte = byte;
    ack.pkt_id = pkt_id;
    return &ack;
}

// CONNACK 组包实现
struct mqtt_connack *mqtt_packet_connack(unsigned char byte,
                                         unsigned char cflags,
                                         unsigned char rc) {
    static struct mqtt_connack connack;
    connack.header.byte = byte;
    connack.byte = cflags;
    connack.rc = rc;
    return &connack;
}

// SUBACK 组包实现
struct mqtt_suback *mqtt_packet_suback(unsigned char byte,
                                       unsigned short pkt_id,
                                       unsigned char *rcs,
                                       unsigned short rcslen) {
    struct mqtt_suback *suback = malloc(sizeof(*suback));
    suback->header.byte = byte;
    suback->pkt_id = pkt_id;
    suback->rcslen = rcslen;
    suback->rcs = malloc(rcslen);
    memcpy(suback->rcs, rcs, rcslen);
    return suback;
}

// PUBLISH 组包实现
struct mqtt_publish *mqtt_packet_publish(unsigned char byte,
                                         unsigned short pkt_id,
                                         size_t topiclen,
                                         unsigned char *topic,
                                         size_t payloadlen,
                                         unsigned char *payload) {
    struct mqtt_publish *publish = malloc(sizeof(*publish));
    publish->header.byte = byte;
    publish->pkt_id = pkt_id;
    publish->topiclen = topiclen;
    publish->topic = topic;
    publish->payloadlen = payloadlen;
    publish->payload = payload;
    return publish;
}

// 释放包资源
void mqtt_packet_release(union mqtt_packet *pkt, unsigned type) {
    switch (type) {
        case CONNECT:
            free(pkt->connect.payload.client_id);
            if (pkt->connect.bits.username == 1)
                free(pkt->connect.payload.username);
            if (pkt->connect.bits.password == 1)
                free(pkt->connect.payload.password);
            if (pkt->connect.bits.will == 1) {
                free(pkt->connect.payload.will_message);
                free(pkt->connect.payload.will_topic);
            }
            break;
        case SUBSCRIBE:
        case UNSUBSCRIBE:
            for (unsigned i = 0; i < pkt->subscribe.tuples_len; i++)
                free(pkt->subscribe.tuples[i].topic);
            free(pkt->subscribe.tuples);
            break;
        case SUBACK:
            free(pkt->suback.rcs);
            break;
        case PUBLISH:
            free(pkt->publish.topic);
            free(pkt->publish.payload);
            break;
        default:
            break;
    }
}
```

# 编码实现

我们接下来处理编码函数，编码函数其实就是解码函数的反方向操作：我们使用内存对象创造一个字节流，之后可以通过socket发出去。

现在我们有一些函数返回指向 `static struct` 的指针（例如上方代码中的 `mqtt_packet_header` ），在单线程的情况下这是没什么问题的。 **在多线程环境下，一定会出问题**，每次这种函数的返回都会指向同一片内存区域，可能导致各种冲突。因此为了将来的改进，需要重构这些部分，使用 `malloc` 来为每次返回分配地址。

我们采用和之前解码函数一样的方式来映射编码函数。做一个静态数组，其中的序号恰好等于包类型。

```c src/mqtt.c

// MQTT 编码函数接口
typedef unsigned char *mqtt_pack_handler(const union mqtt_packet *);

// 编码函数数组, 其中索引和包类型id对应
static mqtt_pack_handler *pack_handlers[13] = {
    NULL,
    NULL,
    pack_mqtt_connack,
    pack_mqtt_publish,
    pack_mqtt_ack,
    pack_mqtt_ack,
    pack_mqtt_ack,
    pack_mqtt_ack,
    NULL,
    pack_mqtt_suback,
    NULL,
    pack_mqtt_ack,
    NULL
};

// header 的编码实现
static unsigned char *pack_mqtt_header(const union mqtt_header *hdr) {
    unsigned char *packed = malloc(MQTT_HEADER_LEN);
    unsigned char *ptr = packed;
    pack_u8(&ptr, hdr->byte);
    // Remaining Length 1byte 值为0
    mqtt_encode_length(ptr, 0);
    return packed;
}

// ACK 的编码实现
static unsigned char *pack_mqtt_ack(const union mqtt_packet *pkt) {
    unsigned char *packed = malloc(MQTT_ACK_LEN); // 4byte
    unsigned char *ptr = packed;
    pack_u8(&ptr, pkt->ack.header.byte);
    mqtt_encode_length(ptr, MQTT_HEADER_LEN); // 这里指还有2byte 内容是 pkt_id
    ptr++; // 因为 mqtt_encode_length 不会移动指针, 只会返回 Remaining Length 的长度, 而这里长度显然为1
    pack_u16(&ptr, pkt->ack.pkt_id);
    return packed;
}

// CONNACK 的编码实现
static unsigned char *pack_mqtt_connack(const union mqtt_packet *pkt) {
    unsigned char *packed = malloc(MQTT_ACK_LEN);
    unsigned char *ptr = packed;
    pack_u8(&ptr, pkt->connack.header.byte);
    mqtt_encode_length(ptr, MQTT_HEADER_LEN);
    ptr++;
    pack_u8(&ptr, pkt->connack.byte);
    pack_u8(&ptr, pkt->connack.rc);
    return packed;
}

// SUBACK 的编码实现
static unsigned char *pack_mqtt_suback(const union mqtt_packet *pkt) {
    // 计算总长度
    size_t pktlen = MQTT_HEADER_LEN + sizeof(uint16_t) + pkt->suback.rcslen;
    unsigned char *packed = malloc(pktlen + 0);
    unsigned char *ptr = packed;
    // 编码固定头
    pack_u8(&ptr, pkt->suback.header.byte);
    // 剩余部分的长度
    size_t len = sizeof(uint16_t) + pkt->suback.rcslen;
    // 变长表示剩余部分长度
    int step = mqtt_encode_length(ptr, len);
    // 指针后移
    ptr += step;
    // 剩余部分编码
    pack_u16(&ptr, pkt->suback.pkt_id);
    for (int i = 0; i < pkt->suback.rcslen; i++)
        pack_u8(&ptr, pkt->suback.rcs[i]);
    return packed;
}

// PUBLISH 的编码实现
static unsigned char *pack_mqtt_publish(const union mqtt_packet *pkt) {
    // pktlen 至少有这么多: 头部至少2byte(1byte头 + 至少1byte的Remaining Length)
    // sizeof(uint16_t) 表示 topiclen 的长度, 因为 payloadlen 是不被编码到字节流中的
    // topiclen 和 payloadlen 的内容
    size_t pktlen = MQTT_HEADER_LEN + sizeof(uint16_t) +
        pkt->publish.topiclen + pkt->publish.payloadlen;
    // 这里是去除 fixed header 之外的内容长度
    size_t len = 0L;
    // qos > 0, 说明有pkt_id, 需要 +2byte
    if (pkt->header.bits.qos > AT_MOST_ONCE)
        pktlen += sizeof(uint16_t);
    // 这里是通过剩余长度计算变长部分还需要的长度, 前面已经预留了1byte
    int remaininglen_offset = 0;
    if ((pktlen - 1) > 0x200000)
        remaininglen_offset = 3;
    else if ((pktlen - 1) > 0x4000)
        remaininglen_offset = 2;
    else if ((pktlen - 1) > 0x80)
        remaininglen_offset = 1;
    // 这里是总包长
    pktlen += remaininglen_offset;
    unsigned char *packed = malloc(pktlen);
    unsigned char *ptr = packed;
    pack_u8(&ptr, pkt->publish.header.byte);
    // 除去 fixed header 之外剩余部分的长度
    len += (pktlen - MQTT_HEADER_LEN - remaininglen_offset);
    // 编码 Remaining Length
    int step = mqtt_encode_length(ptr, len);
    ptr += step;
    // 编码 topiclen 和后续的 topic 内容
    pack_u16(&ptr, pkt->publish.topiclen);
    pack_bytes(&ptr, pkt->publish.topic);
    // 当 QoS > 0 时, 编码 pkt_id
    if (pkt->header.bits.qos > AT_MOST_ONCE)
        pack_u16(&ptr, pkt->publish.pkt_id);
    // 编码 payload 的内容
    pack_bytes(&ptr, pkt->publish.payload);
    return packed;
}

// 编码函数入口
unsigned char *pack_mqtt_packet(const union mqtt_packet *pkt, unsigned type) {
    if (type == PINGREQ || type == PINGRESP)
        return pack_mqtt_header(&pkt->header);
    return pack_handlers[type](pkt);
}
```

# socket 封装

我们计划创建一个单线程 TCP 服务器，使用 **epoll** 接口实现多路 I/O。Epoll 是继 **select** 和 **poll** 之后内核 2.5.44 添加的最新的多路复用机制，也是性能最高、连接数最多的多路复用机制，它在 BSD 和 BSD-like (Mac OSX) 系统中的对应机制是 **kqueue**。

我们需要定义一些函数来管理我们的socket descriptor。

```c src/network.h
#include <stdio.h>
#include <stdint.h>
#include <sys/types.h>
#include "util.h"

// 地址族
#define UNIX    0
#define INET    1

// 设置为 non-blocking 模式
int set_nonblocking(int);

// 将 TCP_NODELAY 设置为 true, 用来关闭 Nagle's algorithm, 关闭收包时的缓冲等待
int set_tcp_nodelay(int);

// 创建 socket 服务的辅助函数
int create_and_bind(const char *, const char *, int);

// 创建一个 non-blocking socket 并监听指定的地址和端口
int make_listen(const char *, const char *, int);

// 接收链接并进行后续处理, 将链接分配到 epollfd
int accept_connection(int);
```

我们定义了一些简单的辅助函数，用来创建和绑定 `socket` 端口，处理新链接并把 `socket` 设置为 `non-blocking` 模式（这样才能发挥 **epoll** 的复用能力）。

我不喜欢必须处理每个进出服务器的字节，在我写的涉及到TCP通信的程序中，我都会定义这两个函数：
- `ssize_t send_bytes(int, const unsigned char *, size_t)` 用于在while循环中持续发送数据，直到把数据全部发送完。正确捕获 `EAGAIN` 或 `EWOUDLBLOCK` 异常。
- `ssize_t recv_bytes(int, unsigned char *, size_t)` 在while循环中获得任意长度的数据。正确捕获 `EAGAIN` 或 `EWOUDLBLOCK` 异常。

```c src/network.h
// I/O 管理函数

// 在循环中发出所有数据, 避免内核buffer可用性造成的中断(EAGAIN EWOUDLBLOCK)
ssize_t send_bytes(int, const unsigned char *, size_t);

// 从 fd 中读取指定长度的数据进入 buffer
ssize_t recv_bytes(int, unsigned char *, size_t);
```

## socket 封装实现

接下来是 `network.c` 的实现。

```c src/network.c
#define _DEFAULT_SOURCE
#include <stdlib.h>
#include <errno.h>
#include <netdb.h>
#include <unistd.h>
#include <fcntl.h>
#include <arpa/inet.h>
#include <sys/un.h>
#include <sys/epoll.h>
#include <sys/timerfd.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/eventfd.h>
#include "network.h"
#include "config.h"

// 设置 non-blocking socket
int set_nonblocking(int fd) {
    int flags, result;
    flags = fcntl(fd, F_GETFL, 0);
    if (flags == -1)
        goto err;
    result = fcntl(fd, F_SETFL, flags | O_NONBLOCK);
    if (result == -1)
        goto err;
    return 0;
err:
    perror("set_nonblocking");
    return -1;
}

// 设置 TCP_NODELAY 用以关闭 Nagle's algorithm
int set_tcp_nodelay(int fd) {
    return setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &(int) {1}, sizeof(int));
}

// UNIX socket 的绑定方法
// return fd
// sockpath 文件路径
static int create_and_bind_unix(const char *sockpath) {
    struct sockaddr_un addr;
    int fd;
    // 创建 socket
    if ((fd = socket(AF_UNIX, SOCK_STREAM, 0)) == -1) {
        perror("socket error");
        return -1;
    }
    // addr初始值全0
    memset(&addr, 0, sizeof(addr));
    // 赋值
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, sockpath, sizeof(addr.sun_path) - 1);

    // 译者没有明白为何 unlink 会出现在此处
    unlink(sockpath);

    // 绑定 socket
    if (bind(fd, (struct sockaddr*) &addr, sizeof(addr)) == -1) {
        perror("bind error");
        return -1;
    }
    return fd;
}

// TCP socket 的绑定方法
// return fd
// host TCP 地址
// port TCP 端口
static int create_and_bind_tcp(const char *host, const char *port) {
    struct addrinfo hints = {
        .ai_family = AF_UNSPEC,       // 不指定协议族, 系统自定可以是IP4 或 IP6
        .ai_socktype = SOCK_STREAM,   // 面向流, 就是TCP
        .ai_flags = AI_PASSIVE        // 被动模式, 可以监听任意地址端口
    };
    // result 是 getaddrinfo 提供的 addrinfo, rp 指如果绑定不成功, 可以变成下一个 addrinfo
    struct addrinfo *result, *rp;
    int sfd;
    if (getaddrinfo(host, port, &hints, &result) != 0) {
        perror("getaddrinfo error");
        return -1;
    }
    for (rp = result; rp != NULL; rp = rp->ai_next) {
        // 先使用 rp 生成 socket
        sfd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
        // 如果失败就下一个 rp
        if (sfd == -1) continue;
        // 设置 SO_REUSEADDR 这样关闭进程后可以重用端口
        if (setsockopt(sfd, SOL_SOCKET, SO_REUSEADDR,
                       &(int) { 1 }, sizeof(int)) < 0)
            perror("SO_REUSEADDR");
        if ((bind(sfd, rp->ai_addr, rp->ai_addrlen)) == 0) {
            // bind 成功
            break;
        }
        // 绑定失败记得关闭 socket
        close(sfd);
    }
    if (rp == NULL) {
        perror("Could not bind");
        return -1;
    }
    freeaddrinfo(result);
    return sfd;
}

// 绑定入口
int create_and_bind(const char *host, const char *port, int socket_family) {
    int fd;
    if (socket_family == UNIX)
        fd = create_and_bind_unix(host);
    else
        fd = create_and_bind_tcp(host, port);
    return fd;
}

// 创建一个 non-blocking socket, 监听指定的地址端口
// return server file descriptor
// host 地址或UNIX path
// port 端口
// socket_family 地址族 AF_UNIX 或 AF_INET
int make_listen(const char *host, const char *port, int socket_family) {
    int sfd;
    if ((sfd = create_and_bind(host, port, socket_family)) == -1)
        abort();
    if ((set_nonblocking(sfd)) == -1)
        abort();
    // 仅当 TCP链接时设置 TCP_NODELAY
    if (socket_family == INET)
        set_tcp_nodelay(sfd);
    // conf是本程序的配置文件
    if ((listen(sfd, conf->tcp_backlog)) == -1) {
        perror("listen");
        abort();
    }
    return sfd;
}

// 接收链接后的处理
// return 客户端 fd
// serversock 服务端fd
int accept_connection(int serversock) {
    int clientsock;
    struct sockaddr_in addr;
    socklen_t addrlen = sizeof(addr);
    if ((clientsock = accept(serversock,
                             (struct sockaddr *) &addr, &addrlen)) < 0)
        return -1;
    set_nonblocking(clientsock);

    // 仅当 TCP链接时设置 TCP_NODELAY
    if (conf->socket_family == INET)
        set_tcp_nodelay(clientsock);
    char ip_buff[INET_ADDRSTRLEN + 1];
    // 将ip地址转为文本, 这里用作检查客户端地址
    if (inet_ntop(AF_INET, &addr.sin_addr,
                  ip_buff, sizeof(ip_buff)) == NULL) {
        close(clientsock);
        return -1;
    }
    return clientsock;
}

// 向 fd 发送指定长度的数据
// return 成功发送的数据长度
// fd 发送数据的目的
// buf 发送数据内容地址
// len 需要发送的数据长度
ssize_t send_bytes(int fd, const unsigned char *buf, size_t len) {
    // 发送数据的总长度
    size_t total = 0;
    // 剩余需要发送数据的长度
    size_t bytesleft = len;
    // 单次发送数据长度
    ssize_t n = 0;
    while (total < len) {
        // 发送 bytesleft 长度的数据
        n = send(fd, buf + total, bytesleft, MSG_NOSIGNAL);
        if (n == -1) {
            // 当 fd 被阻塞时, 直接返回已经发送的长度
            if (errno == EAGAIN || errno == EWOULDBLOCK)
                break;
            else
                goto err;
        }
        total += n;
        bytesleft -= n;
    }
    return total;
err:
    fprintf(stderr, "send(2) - error sending data: %s", strerror(errno));
    return -1;
}

// 从 fd 中获得指定长度的数据
// retrun 成功读取的长度 -1 表示异常
// fd 数据源
// buf 存放结果的指针
// bufsize 期望读取的数据长度
ssize_t recv_bytes(int fd, unsigned char *buf, size_t bufsize) {
    // 单次获取的数据长度
    ssize_t n = 0;
    // 获取的总数据长度
    ssize_t total = 0;
    while (total < (ssize_t) bufsize) {
        // 使用 recv 函数获得最大 bufsize - total 的数据
        if ((n = recv(fd, buf, bufsize - total, 0)) < 0) {
            // fd被阻塞了, 此时total的返回也许是小于 bufsize 的值, 调用者可以选择重试
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                break;
            } else
                // 对于其他的异常则报错
                goto err;
        }
        if (n == 0)
            return 0;
        buf += n;
        total += n;
    }
    return total;
err:
    fprintf(stderr, "recv(2) - error reading data: %s", strerror(errno));
    return -1;
}
```

# epoll 封装

为了让 **epoll** API能够更加简单易用。我对 epoll 进行了一些的封装，让我们就可以通过注册回调函数的方式来响应事件。

网络上有很多使用 epoll 的示例，大部分都是描述基本用法：注册一个 socket 并启动一个循环来监听事件，每当 socket 需要被读写时，调用一个函数来使用它们。这些例子当然简单好用，但是并没有告诉我们如何通过回调的方式使用 epoll。经过思考后，我发现可以使用 `epoll_event` 自带的 `epoll_data` 来解决这个问题：

```c
typedef union epoll_data {
   void        *ptr;
   int          fd;
   uint32_t     u32;
   uint64_t     u64;
} epoll_data_t;
```

正如你看到的，`epoll_data` 中有一个 `void *`，一个常常用来保存fd的 `int`，还有两个大小不同的 `uint`。我计划做一个自定义事件结构体，其中包括了fd、一些自定义数据和最关键的回调函数指针。然后我们可以把自定义事件结构体绑定到 `epoll_data` 的 `void *` 中，如此一来，每当事件发生时，我们都可以通过 `epoll_data` 获得所有我们需要的东西。

我想要定义两种类型的回调，一种是事件触发的回调，另一种是间隔触发的周期性回调。我们需要把 epoll 封装到一个自定义结构里，来实现这两种回调。对于这两种回调的处理，我们则会采用完全相同的方式：获得 `epoll_data`，在其中获得所有我们所需的数据和需要执行的回调函数。

**接收数据包并使用 epoll_wait 处理的顺序图**
![Epoll sequential diagram](epoll-sequential.png)

我们需要定义两种结构体和一种函数指针

- **struct evloop** 封装 epoll 实例的结构体，添加了各种参数用来实现我们的业务设计
- **struct closure** 上文中提到的自定义事件结构体，封装了各种事件参数和回调函数的指针
- **void callback(struct evloop *, void *)** 回调函数的接口，在 **closure** 里真正被执行的函数的接口

另外，我们需要在 .c 文件中实现一些对 `evloop` 的创建、删除和管理功能。

```c src/network.h
// epoll 的业务包装，包括 epoll 实例本身和其他参数
// 使用 EPOLLONESHOT 处理事件，并且每次都需要手动重置，这样可以保证未来适应多线程架构
struct evloop {
    int epollfd;                // epoll 实例fd
    int max_events;             // 单次处理事件最大数量
    int timeout;                // 事件等待超时事件
    int status;                 // 运行状态(是否运行中)
    struct epoll_event *events; // 事件数组, 用来接收 epoll_wait 获得的一组并发事件
    // 周期性任务控制相关
    int periodic_maxsize;       // 周期性任务数组初始大小
    int periodic_nr;            // 当前周期性任务数量
    struct {
        int timerfd;
        struct closure *closure;
    } **periodic_tasks;         // 周期性任务列表 timerfd <-> closure
} evloop;

// 回调函数接口
typedef void callback(struct evloop *, void *);

// 自定义事件结构体
struct closure {
    int fd;                     // 监听的 fd
    void *obj;                  // 存放一些需要的自定义数据
    void *args;                 // 可以被callback使用的参数, 指向用户自定义结构, 实际调用时就是 call 的第二个参数
    char closure_id[UUID_LEN];  // closure 的 UUID
    struct bytestring *payload; // callback 的结果, 可以被网络发送的数据流
    callback *call;             // 会被执行的回调函数
};

// evloop 的创建、初始化、销毁函数
struct evloop *evloop_create(int, int);
void evloop_init(struct evloop *, int, int);
void evloop_free(struct evloop *);

// 一个阻塞的循环, 监听各种触发并执行对应的回调
int evloop_wait(struct evloop *);

// 添加一个 closure, 其中包含一个回调函数
// 回调函数是单次触发的(边沿触发), 但是每次触发后都会被重置, 这样下次依然可以触发
void evloop_add_callback(struct evloop *, struct closure *);

// 添加一个周期性的 closure, 间隔指定事件触发
void evloop_add_periodic_task(struct evloop *,
                              int,
                              unsigned long long,
                              struct closure *);

// 注销一个 closure, 删除对其 fd 的监听
int evloop_del_callback(struct evloop *, struct closure *);

// 重置该 closure 对 read 事件的监听
int evloop_rearm_callback_read(struct evloop *, struct closure *);

// 重置该 closure 对 write 事件的监听
int evloop_rearm_callback_write(struct evloop *, struct closure *);

// 以下三个函数是对 epoll 原始API的封装, 供上方的函数调用
// EPOLL_CTL_ADD 的封装, 向 epoll 添加监听
int epoll_add(int, int, int, void *);

// EPOLL_CTL_MOD 的封装, 可以重置 EPOLLONESHOT, 让 closure 下次仍被触发
int epoll_mod(int, int, int, void *);

// EPOLL_CTL_DEL 的封装, 删除对某个 fd 的监听
int epoll_del(int, int);
```

## epoll 封装实现

在头文件中定义了我们网络所需的各种工具函数后，接下来我们开始进行函数实现。

让我们先从最简单的开始，`evloop` 实例的创建、初始化和删除。他包括了这些内容：
- `epoll` 的 `fd` 即 `epollfd`
- 单次处理的最大事件数量
- 一个毫秒单位的超时时间
- loop是否正在运行的状态标识
- 动态大小的周期性任务数组

```c src/network.c
/******************************
 *         EPOLL APIS         *
 ******************************/
#define EVLOOP_INITIAL_SIZE 4 // 默认周期任务数组大小

// 创建并初始化 evloop
struct evloop *evloop_create(int max_events, int timeout) {
    struct evloop *loop = malloc(sizeof(*loop));
    evloop_init(loop, max_events, timeout);
    return loop;
}

void evloop_init(struct evloop *loop, int max_events, int timeout) {
    loop->max_events = max_events;
    loop->events = malloc(sizeof(struct epoll_event) * max_events);
    loop->epollfd = epoll_create1(0);  // 这里创建 epoll 实例
    loop->timeout = timeout;
    loop->periodic_maxsize = EVLOOP_INITIAL_SIZE;
    loop->periodic_nr = 0;
    loop->periodic_tasks = malloc(EVLOOP_INITIAL_SIZE * sizeof(*loop->periodic_tasks));
    loop->status = 0;
}

// 释放 evloop
void evloop_free(struct evloop *loop) {
    free(loop->events);
    for (int i = 0; i < loop->periodic_nr; i++)
        free(loop->periodic_tasks[i]);
    free(loop->periodic_tasks);
    free(loop);
}
```

接着，我们需要实现三个包装 `epoll` API的函数，用来创建、修改和删除 `epoll` 对 `fd` 的监听。我们封装函数的目的是为所有的 `epoll` 监听都添加 `EPOLLET` 和 `EPOLLONESHOT` 标识。`EPOLLET` 标识可以让 `epoll` 工作在`边沿触发`模式，`EPOLLONESHOT` 标识则可以确保 `epoll` 对某个事件触发仅产生一次（然后我们通过手动重置的方式让其可以继续响应）。

这样的设置可以避免未来我们在使用多线程架构时，一次事件的传入会唤醒所有等待中的线程，这被称为`惊群效应`(thundering herd problem)，不过这些都是后话，暂时可以不用深究。

```c src/network.c
// 添加监听
// return 添加结果
// efd file descriptor
// fd 被监听的 fd
// evs 被监听的事件(可以是一个或一组)
// data 传入自定义结构体
int epoll_add(int efd, int fd, int evs, void *data) {
    struct epoll_event ev;
    // 在 epoll_data 中设置 fd
    ev.data.fd = fd;
    // 注意 epoll_data 是 union, 如果有data并在此处设置, 那么上一行的 ev.data.fd 就不能再使用(是随机数)
    if (data)
        ev.data.ptr = data;
    // 将所有事件都设置为 边沿触发(EPOLLET) 和 触发后取消监听(EPOLLONESHOT)
    ev.events = evs | EPOLLET | EPOLLONESHOT;
    return epoll_ctl(efd, EPOLL_CTL_ADD, fd, &ev);
}

// 修改监听 主要目的是让触发过的事件可以再次被触发
int epoll_mod(int efd, int fd, int evs, void *data) {
    struct epoll_event ev;
    ev.data.fd = fd;
    // Being ev.data a union, in case of data != NULL, fd will be set to random
    if (data)
        ev.data.ptr = data;
    ev.events = evs | EPOLLET | EPOLLONESHOT;
    // 通过 EPOLL_CTL_MOD 可以让事件再次能被触发
    return epoll_ctl(efd, EPOLL_CTL_MOD, fd, &ev);
}

// 删除监听
int epoll_del(int efd, int fd) {
    return epoll_ctl(efd, EPOLL_CTL_DEL, fd, NULL);
}
```

这里有两件事需要注意：

- 第一，如前所述，`epoll_event` 中包括了一个 `union epoll_data`，其中可以保存一个 `fd` **或** 一个 `void *`。我们选择了使用后者，传入了我们的 `closure`，这其中包含了更多有用的信息，也包括 `fd` 在内。

- 第二，刚才我们定义的添加和修改函数的第三个参数，可以接收一组事件，一般而言是 `EPOLLIN` 或 `EPOLLOUT`。同时我们添加了 `EPOLLONESHOT` 标识，这意味着当事件触发一次后就不会再次触发，除非我们手动重置该事件。这样做是为了保持对低级事件触发的某种程度的控制，并为将来的多线程实现留出空间。这篇[文档](https://idea.popcount.org/2017-02-20-epoll-is-fundamentally-broken-12/)精彩地阐述了 `epoll` 这种设计的好处，以及为什么最好使用 `EPOLLONESHOT` 标志。

## epoll 循环实现

我们继续实现我们的封装，接下来是一些回调函数的注册、周期回调的注册以及主循环。

```c src/network.c
// 添加回调
// loop loop封装实例
// cb 自定义事件封装 closure
void evloop_add_callback(struct evloop *loop, struct closure *cb) {
    if (epoll_add(loop->epollfd, cb->fd, EPOLLIN, cb) < 0)
        perror("Epoll register callback: ");
}

// 添加周期事件
// loop loop封装实例
// seconds 以秒为单位的到期时间或触发周期
// ns 以纳秒为单位的到期时间或触发周期
// cb 自定义事件封装
void evloop_add_periodic_task(struct evloop *loop,
                              int seconds,
                              unsigned long long ns,
                              struct closure *cb) {
    // 表示时间间隔或时间点的结构
    struct itimerspec timervalue;
    int timerfd = timerfd_create(CLOCK_MONOTONIC, 0);
    memset(&timervalue, 0x00, sizeof(timervalue));
    // 设置初始的到期时间 (多久后执行
    timervalue.it_value.tv_sec = seconds;
    timervalue.it_value.tv_nsec = ns;
    // 设置初始的触发周期 (间隔多久执行
    timervalue.it_interval.tv_sec = seconds;
    timervalue.it_interval.tv_nsec = ns;
    // 设置好 timer
    if (timerfd_settime(timerfd, 0, &timervalue, NULL) < 0) {
        perror("timerfd_settime");
        return;
    }
    // 将 timer 添加到 epoll, 让其能够触发
    struct epoll_event ev;
    ev.data.fd = timerfd;
    ev.events = EPOLLIN;
    if (epoll_ctl(loop->epollfd, EPOLL_CTL_ADD, timerfd, &ev) < 0) {
        perror("epoll_ctl(2): EPOLLIN");
        return;
    }
    // 将周期性任务的信息绑定到 loop
    // 如果周期性任务的数量大于periodic_maxsize, 动态扩容
    if (loop->periodic_nr + 1 > loop->periodic_maxsize) {
        loop->periodic_maxsize *= 2;
        loop->periodic_tasks =
            realloc(loop->periodic_tasks,
                    loop->periodic_maxsize * sizeof(*loop->periodic_tasks));
    }
    // 存储周期性任务的内容 timerfd 和 自定义事件
    loop->periodic_tasks[loop->periodic_nr] =
        malloc(sizeof(*loop->periodic_tasks[loop->periodic_nr]));
    loop->periodic_tasks[loop->periodic_nr]->closure = cb;
    loop->periodic_tasks[loop->periodic_nr]->timerfd = timerfd;
    // 记录当前绑定了多少周期性任务
    loop->periodic_nr++;
}

// epoll 主循环
int evloop_wait(struct evloop *el) {
    int rc = 0;             // 返回值
    int events = 0;         // 单次触发事件数
    long int timer = 0L;    // 拿到我们周期性事件的 timerfd
    int periodic_done = 0;  // 标记是否是周期性事件并且已经执行
    while (1) {
        // 等待事件发生
        events = epoll_wait(el->epollfd, el->events,
                            el->max_events, el->timeout);
        // 有异常
        if (events < 0) {
            // 系统中断, 暂时不管
            if (errno == EINTR)
                continue;
            // 确实出了问题, 结束循环
            rc = -1;
            el->status = errno;
            break;
        }
        // 循环处理每个事件
        for (int i = 0; i < events; i++) {
            // 错误校验 检查是否是错误事件 检查是否不是输入输出事件
            if ((el->events[i].events & EPOLLERR) ||
                (el->events[i].events & EPOLLHUP) ||
                (!(el->events[i].events & EPOLLIN) &&
                 !(el->events[i].events & EPOLLOUT))) {
                // 总之这个 fd 上出现了一些异常, 我们把链接关了
                perror ("epoll_wait(2)");
                shutdown(el->events[i].data.fd, 0);
                close(el->events[i].data.fd);
                el->status = errno;
                continue;
            }
            // 拿到我们的 closure
            struct closure *closure = el->events[i].data.ptr;
            // 标记没有完成周期事件
            periodic_done = 0;
            // 当没有被标识完成时, 循环查找我们存储的周期事件
            for (int i = 0; i < el->periodic_nr && periodic_done == 0; i++) {
                // 找到了
                if (el->events[i].data.fd == el->periodic_tasks[i]->timerfd) {
                    // 拿到 closure
                    struct closure *c = el->periodic_tasks[i]->closure;
                    // 读 timerfd
                    (void) read(el->events[i].data.fd, &timer, 8);
                    // 执行回调
                    c->call(el, c->args);
                    // 标记完成
                    periodic_done = 1;
                }
            }
            if (periodic_done == 1)
                continue;
            // 并不是完成了某个周期性事件 那就是触发事件了 这里执行回调
            closure->call(el, closure->args);
        }
    }
    return rc;
}

// 重置该 closure 对 read 事件的监听
int evloop_rearm_callback_read(struct evloop *el, struct closure *cb) {
    return epoll_mod(el->epollfd, cb->fd, EPOLLIN, cb);
}

// 重置该 closure 对 write 事件的监听
int evloop_rearm_callback_write(struct evloop *el, struct closure *cb) {
    return epoll_mod(el->epollfd, cb->fd, EPOLLOUT, cb);
}

// 删除回调函数
int evloop_del_callback(struct evloop *el, struct closure *cb) {
    return epoll_del(el->epollfd, cb->fd);
}
```

在我们之前的所有代码中，`evloop_wait` 是最有意思的，他启动一个循环不停监视 `epoll_wait`，执行错误检查，区分本次触发是周期性的自动触发或是读写触发，然后执行我们设置的回调函数。

# 结尾

我们的代码越写越多，这次我们又添加了一个模块。

此时我们的文件结构是这样的：

```text
sol/
 ├── src/
 │    ├── mqtt.h
 |    ├── mqtt.c
 │    ├── network.h
 │    ├── network.c
 │    ├── pack.h
 │    └── pack.c
 ├── CHANGELOG
 ├── CMakeLists.txt
 ├── COPYING
 └── README.md
```
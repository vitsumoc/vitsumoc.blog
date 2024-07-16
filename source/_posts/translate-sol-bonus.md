---
title: "[翻译]Sol - 从零开始的MQTT broker - 特别篇：重构与事件循环"
url: translate-sol-bonus
date: 2024-01-03 09:14:03
categories:
- MQTT
tags:
- MQTT
- 翻译
- 网络编程
- 物联网
- C
---

> 原文 [Sol - An MQTT broker from scratch. Refactoring & eventloop](https://codepr.github.io/posts/sol-mqtt-broker-bonus/)

<!--more-->

***更新日期: 2020-02-07***

# 前言

在前面的六个部分中，我们探索了一些常见的 CS 主题，例如网络编程、数据结构，这段短暂的旅程的终点是得到了一个充满了BUG但是勉强可用的MQTT broker。

由于好奇心，我想测试一下我们的项目离真正的生产项目有多么接近，而且我想对项目进行一些重构，减少一些临时的代码，让项目的结构更加合理，同时关注项目的可移植性。

我不会把所有的重构过程都写到博客中，因为那会非常无聊，我只会突出一些最重要的部分，剩下的部分你可以直接把 `master` 分支合并到 `tutorial` 来查看，或者直接克隆 `master` 分支。

首先我按照有限度列出了需要优化的要点：

- 低层的 I/O 处理器，用以正确处理数据流读写
- 对 EPOLL 进行抽象，因为他是 Linux 独有功能，提供一些备选方案
- 管理加密消息，实现可用明文消息或加密消息的透明接口
- 正确处理客户端会话，实现类似 `'+'` 通配符之类的其他 MQTT 功能

*备注：虽然我们自己做的哈希表运行的不错，但我还是决定选择使用久经沙场的 `UTHASH` 库。由于他只有一个头文件，集成进我们的项目也非常容易。他的项目文档在[这里](https://troydhanson.github.io/uthash/)。*

# TCP分片问题

第一个也是最需要被检查的问题是网络通信，在本地进行负载测试时，我发现当负载量较大时程序开始丢包，或者说，内核缓冲区被淹没并开始对数据流进行分片。TCP 作为一个流协议，在处理数据中进行分片是无可厚非的，没有在一开始时就考虑这个问题显然是我比较幼稚，或者说因为我着急写一个可以运行的程序，忽略了底层细节。无论如何，这让程序产生了一些问题，例如解析错误的数据包，或者分片部分被当作数据包的第一个字节，识别成了各种不同的指令等等。

因此，最重要的修复之一是 **server.c** 模块中的 `recv_packet` 函数，特别是为每个客户端添加了类似状态机的行为，使其可以正确执行非阻塞读写，而不会阻塞线程。

我还将应用程序的核心部分，特别是 MQTT 抽象（例如客户端会话和主题）移到了 `sol_internal.h` 中。

```c sol_internal.h
// 客户端行为可以被视为拥有四个状态的状态机：
// - WAITING_HEADER 基础状态, 等待到来的第一字节数据头
// - WAITING_LENGTH 第二个状态, 收到了头部但还没有收取全部的 remaing data
// - WAITING_DATA 第三个状态, 基于 remaing data 判断还有多少剩余数据
// - SENDING_DATA 最后一个状态, 已经收取了全部的数据包, 接下来判断是否需要返回数据包
enum client_status {
    WAITING_HEADER,
    WAITING_LENGTH,
    WAITING_DATA,
    SENDING_DATA
};

// 客户端链接的包装类, 客户端可以是订阅者或发布者, 可以拥有会话
// 现在不再需要为每个客户端申请内存, 我在程序启动时初始化了一个客户端池, 当然, 读写 buffer 是使用时再申请的
// 这是一个可以被哈希的结构, 参考 https://troydhanson.github.io/uthash/userguide.html
struct client {
    struct ev_ctx *ctx; // 事件循环上下文指针
    int rc;     // 持有处理的上一个消息的返回码
    int status; // 当前状态
    int rpos;   // 表示去除 Fixed Header, 数据包实际开始的位置
                // 因为收包时需要解析 Fixed Header 中变长的 Remaing Length, 不想在解包时再次解析
                // 就通过此字段记录
    size_t read;  // 已经读取的字节数
    size_t toread;// 完成此数据包总共需要读取的字节数
    unsigned char *rbuf; // 读取 buffer
    size_t wrote; // 已经写入的字节数
    size_t towrite; // 还需写入的字节数
    unsigned char *wbuf;  // 写入 buffer
    char client_id[MQTT_CLIENT_ID_LEN]; // MQTT 规范中的客户端 ID
    struct connection conn; // 网络连接封装, 通过抽象接口支持普通连接或TLS连接
    struct client_session *session; // 客户端会话
    unsigned long last_seen;  // 客户端上次活动的时间戳
    bool online;  // 在线标识
    bool connected; // 是否已经处理 CONNECT 包的标识
    bool has_lwt; // 表示 CONNECT 包是否包含遗嘱 LWT（Last Will and Testament）
    bool clean_session; // 表示是否设置了 clean_session 标识
    UT_hash_handle hh; // UTHASH handle 处理器, 使用 UTHASH 的条件
};

// 每个客户端都持有一个会话, 用来缓存该客户端订阅的主题、失联时错过的消息(只有当 clean_session 为 false)、还有服务器已经发往客户端但没收到回复的消息(inflight messages)(这些消息都带有 message ID)
// 基于MQTT协议, 最大的 mid (message ID) 数量为 65535, 所以 i_acks, i_msgs 和 in_i_acks 被初始化为这个尺寸
// 这是一个可被哈希的结构体, APP可以追踪他完整的生命周期
struct client_session {
    int next_free_mid;                    // 下一个可用的 mid (message ID)
    List *subscriptions;                  // 客户端订阅的所有主题, 使用主题结构体存储
    List *outgoing_msgs;                  // 断开链接期间发往客户端的消息, 使用 mqtt_packet 指针存储
    bool has_inflight;                    // 表示是否有 inflight 消息的标识
    bool clean_session;                   // clean_session 标识
    char session_id[MQTT_CLIENT_ID_LEN];  // session 中引用的 client_id
    struct mqtt_packet lwt_msg;           // 遗嘱消息, 由 CONNECT 设置, 可为空
    struct inflight_msg *i_acks;          // 需要被清理的离线ACK
    struct inflight_msg *i_msgs;          // 由于发送超时, 需要被重传的离线消息
    struct inflight_msg *in_i_acks;       // 需要被客户端清理的离线输入ACK
    UT_hash_handle hh;                    // UTHASH handle 处理器, 使用 UTHASH 的条件
    struct ref refcount;                  // 被引用计数, 用来共享此结构体
};
```

因此，客户端结构现在更加健壮，它存储每个数据包读写的状态，以便在内核空间出现 `EAGAIN` 错误时恢复。

```c server.c
// 客户端接收数据包
static ssize_t recv_packet(struct client *c) {
    ssize_t nread = 0;
    unsigned opcode = 0, pos = 0;
    unsigned long long pktlen = 0LL;
    // 基础状态, 读头部
    if (c->status == WAITING_HEADER) {
        // 读取最初的2byte, 第一个byte应包含消息类型码
        nread = recv_data(&c->conn, c->rbuf + c->read, 2 - c->read);
        // 异常视为断链
        if (errno != EAGAIN && errno != EWOULDBLOCK && nread <= 0)
            return -ERRCLIENTDC;
        // 不管是否全部读取完成, 记录已经读取的数量
        c->read += nread;
        // 没有完全读取, 返回 EAGAIN
        if (errno == EAGAIN && c->read < 2)
            return -ERREAGAIN;
        // 完成进入下一阶段
        c->status = WAITING_LENGTH;
    }
    // 头部已经读完, 已经了解消息类型, 接下来我们读取第2-4byte, 从第一个字节之后的三个字节可能会用来存储包长度
    // 当然, 除了 PINGRESP/PINGREQ 或 DISCONNECT, 他们没有 remaining length
    if (c->status == WAITING_LENGTH) {
        if (c->read == 2) {
            opcode = *c->rbuf >> 4;
            // 数据包类型错误
            if (DISCONNECT < opcode || CONNECT > opcode)
                return -ERRPACKETERR;
            // 数据包类型是 PINGRESP/PINGREQ 或 DISCONNECT, 无需后续处理(没有 remaining length)
            if (opcode > UNSUBSCRIBE) {
                c->rpos = 2;
                c->toread = c->read;
                goto exit;
            }
        }
        // 总共读取至 4 byte
        // 译者觉得这里应该到 5
        nread = recv_data(&c->conn, c->rbuf + c->read, 4 - c->read);
        if (errno != EAGAIN && errno != EWOULDBLOCK && nread <= 0)
            return -ERRCLIENTDC;
        c->read += nread;
        if (errno == EAGAIN && c->read < 4)
            return -ERREAGAIN;
        // 通过 remaining length 获得剩余部分的长度
        pktlen = mqtt_decode_length(c->rbuf + 1, &pos);
        // 超长异常
        if (pktlen > conf->max_request_size)
            return -ERRMAXREQSIZE;
        // rpos 定位到头部和变长长度之后
        c->rpos = pos + 1;
        // 数据包总大小
        c->toread = pktlen + pos + 1;  // pos = bytes used to store length
        // ACK 包无需继续读取
        if (pktlen <= 4)
            goto exit;
        c->status = WAITING_DATA;
    }
    // 读取完整的数据包字节
    nread = recv_data(&c->conn, c->rbuf + c->read, c->toread - c->read);
    if (errno != EAGAIN && errno != EWOULDBLOCK && nread <= 0)
        return -ERRCLIENTDC;
    c->read += nread;
    if (errno == EAGAIN && c->read < c->toread)
        return -ERREAGAIN;
exit:
    return 0;
}

// 在接收链接或回复消息后使用此函数获取后续客户端输入的数据
static inline int read_data(struct client *c) {
    // 我们必须接收一个完整的数据包
    int err = recv_packet(c);
    // 链接断开或收到了错误的数据包
    // TODO：设置一个处理 ERRMAXREQSIZE 的函数, 显示的提醒客户端故障
    if (err < 0)
        goto err;
    // 表示阻塞, 需要继续读取
    if (c->read < c->toread)
        return -ERREAGAIN;
    // 记录
    info.bytes_recv += c->read;
    return 0;
    // 断开链接或故障
err:
    return err;
}

// 通过网络连接向客户端发送数据流, 持续发送直到所有数据发送完成, 通过 towrite 字段跟踪
// 当阻塞时返回 EAGAIN
static inline int write_data(struct client *c) {
    ssize_t wrote = send_data(&c->conn, c->wbuf+c->wrote, c->towrite-c->wrote);
    if (errno != EAGAIN && errno != EWOULDBLOCK && wrote < 0)
        return -ERRCLIENTDC;
    c->wrote += wrote > 0 ? wrote : 0;
    if (c->wrote < c->towrite && errno == EAGAIN)
        return -ERREAGAIN;
    // 发送成功 更新状态
    info.bytes_sent += c->towrite;
    // 重置记录数据
    c->towrite = c->wrote = 0;
    return 0;
}
```

# 加密通讯

需要注意的是，`recv_packet` 和 `write_data` 是两个在 **network.h** 模块中定义的函数：

- `ssize_t send_data(struct connection *, const unsigned char *, size_t)`
- `ssize_t recv_data(struct connection *, unsigned char *, size_t)`

他们都需要使用 `struct connection` 作为第一个参数，后面两个参数就是常规的读/写 buffer 和读/写字节数。

这个连接结构直接针对了前言中需改进列表内的第三条（明文消息和加密消息的抽象），他是客户端链接的抽象实现，并且提供了管理通信所需的4个基本回调函数：

- accept
- send
- recv
- close

这个改进允许我们基于选择的类型创建每条链接，不论是普通链接还是TLS链接都使用相同的函数收发数据。

结构定义如下：

```c network.h
// 链接抽象结构，向外提供统一接口，根据传输层加密与否设置正确的回调函数
// 四个主要的回调函数表示了可以在链接上进行的四种操作：
// - accept
// - read
// - write
// - close
// 同时维护了 ip:port 信息
struct connection {
    int fd;
    SSL *ssl;
    SSL_CTX *ctx;
    char ip[INET_ADDRSTRLEN + 6];
    int (*accept) (struct connection *, int);
    ssize_t (*send) (struct connection *, const unsigned char *, size_t);
    ssize_t (*recv) (struct connection *, unsigned char *, size_t);
    void (*close) (struct connection *);
};
```

结构体中存储了 `SSL *` 和 `SSL_CTX *`，当我们使用普通链接时他们会为 `NULL`。

# 编解码与辅助函数

另一个有益的提升是修正了之前错误的编码和解码函数（感谢[beej networking guide](https://beej.us/guide/bgnet/html/single/bgnet.html#serialization)，这个教程真的很优秀）并且添加了一些工具函数用来处理整形和bytes的解码。

```c pack.c
// 整数解码
long long unpack_integer(unsigned char **buf, char size) {
    long long val = 0LL;
    switch (size) {
        case 'b':
            val = **buf;
            *buf += 1;
            break;
        case 'B':
            val = **buf;
            *buf += 1;
            break;
        case 'h':
            val = unpacki16(*buf);
            *buf += 2;
            break;
        case 'H':
            val = unpacku16(*buf);
            *buf += 2;
            break;
        case 'i':
            val = unpacki32(*buf);
            *buf += 4;
            break;
        case 'I':
            val = unpacku32(*buf);
            *buf += 4;
            break;
        case 'q':
            val = unpacki64(*buf);
            *buf += 8;
            break;
        case 'Q':
            val = unpacku64(*buf);
            *buf += 8;
            break;
    }
    return val;
}

unsigned char *unpack_bytes(unsigned char **buf, size_t len) {
    unsigned char *dest = malloc(len + 1);
    memcpy(dest, *buf, len);
    dest[len] = '\0';
    *buf += len;
    return dest;
}
```

# 微型的事件循环：ev

在单线程环境中抽象主机提供的多路复用API并不是一件困难的事，本质上就是提供一个数据结构，用来持有一组自定义事件。头文件里描述的很清楚，最重要的部分是我们对事件类型的枚举（`enum ev_type`），自定义事件（`struct ev`）和持有自定义事件的数组（`events_monitored`）。这些构成了我们的事件封装（`ev_ctx`）。

`ev_ctx` 中使用不透明的 `void *` 指针可以让我们引用系统提供的任何底层 API，无论是 `EPOLL`、`SELECT` 还是 `KQUEUE`。

```c ev.h
#include <sys/time.h>
#define EV_OK  0
#define EV_ERR 1

// 事件类型, 支持或运算
enum ev_type {
    EV_NONE       = 0x00,
    EV_READ       = 0x01,
    EV_WRITE      = 0x02,
    EV_DISCONNECT = 0x04,
    EV_EVENTFD    = 0x08,
    EV_TIMERFD    = 0x10,
    EV_CLOSEFD    = 0x20    // 停止循环, 关闭服务
};
struct ev_ctx;

// 自定义事件, 存储与事件上下文的数组中
// 携带有客户端信息, 被触发时执行对应的回调函数
struct ev {
    int fd;
    int mask;
    void *rdata; // 读取回调函数参数的不透明指针
    void *wdata; // 写入回调函数参数的不透明指针
    void (*rcallback)(struct ev_ctx *, void *); // 读取回调函数
    void (*wcallback)(struct ev_ctx *, void *); // 写入回调函数
};

// 事件循环上下文结构, 持有被监视的事件对象和指向后端事件引擎的指针
// 当前我们仍然使用 epoll, 因为现在的线程模型与 select 默认的电平触发机制不是很适配
// 对于单线程场景, 抽象select很容易
// 现在由于 epoll 边缘触发 + 单次触发 机制，我们可以轻松的在多线程场景使用我们的事件循环
struct ev_ctx {
    int events_nr;
    int maxfd;                          // 最大监听fd数, events_monitored 的长度不得小于此数
    int stop;
    int maxevents;
    unsigned long long fired_events;    // 被触发事件数
    struct ev *events_monitored;        // 监控事件列表
    void *api;                          // 指向基于平台的事件引擎的指针
};
void ev_init(struct ev_ctx *, int);
void ev_destroy(struct ev_ctx *);

// 轮询 ev_ctx 中的事件, 无限阻塞或超时, 当有事件需要处理时返回
int ev_poll(struct ev_ctx *, time_t);

// 调用 ev_poll 再阻塞中轮询事件, 每轮中执行事件中对应的回调函数
int ev_run(struct ev_ctx *);

// 触发停止事件
void ev_stop(struct ev_ctx *);

// 向循环队列尾部添加 fd, 和 ev_fire_event 相同只是没有回调函数
// 可以用来添加 socket 监听之类的简单描述符
int ev_watch_fd(struct ev_ctx *, int, int);

// 在循环中删除 fd, 虽然 close 调用足以从事件引擎中删除 fd, 但是还是用此调用封装来确保所有相关事件都被清理并设置为 EV_NONE
int ev_del_fd(struct ev_ctx *, int);

// 注册一个新事件, 在功能上他和 ev_fire_event 相同但是此函数用于注册一个还未加入事件监听的fd
// 此函数可以被集成到 ev_fire_event 中, 但是我还是倾向于保持语义分离
int ev_register_event(struct ev_ctx *, int, int,
                      void (*callback)(struct ev_ctx *, void *), void *);

// 注册一个周期性事件
int ev_register_cron(struct ev_ctx *,
                     void (*callback)(struct ev_ctx *, void *),
                     void *,
                     long long, long long);

// 为下一个循环周期的 FD 注册一个新事件
// 和 ev_watch_fd相同, 但可以携带回调函数和参数
int ev_fire_event(struct ev_ctx *, int, int,
                  void (*callback)(struct ev_ctx *, void *), void *);
```

在服务器初始化时，`ev_ctx` 会被注册一些基本的周期性事件和服务端口的 `on_accpet` 事件。之后我们的程序就由事件循环不停驱动，比如当客户端链接建立后，我们会对输入的数据进行监听，触发 `read_callback`，收到完整的数据包并处理后，决定是否要发送回复。

```text
                             MAIN THREAD
                              [EV_CTX]

    ACCEPT_CALLBACK         READ_CALLBACK         WRITE_CALLBACK
  -------------------    ------------------    --------------------
           |                     |                       |
        ACCEPT                   |                       |
           | ------------------> |                       |
           |               READ AND DECODE               |
           |                     |                       |
           |                     |                       |
           |                  PROCESS                    |
           |                     |                       |
           |                     |                       |
           |                     | --------------------> |
           |                     |                     WRITE
        ACCEPT                   |                       |
           | ------------------> | <-------------------- |
           |                     |                       |
```

这是一个连接客户端的生命周期，我们有一个 `accept` 回调函数，他将接入的链接放入事件循环中，并且开启读取监听：

```c server.c
// 处理输入的链接, 创建一个客户端对象并关联到fd
// 设置为 EV_READ 事件并绑定 read_callback 用以处理输入的数据流
static void accept_callback(struct ev_ctx *ctx, void *data) {
    int serverfd = *((int *) data);
    while (1) {
        // 接收一个新的连接, 将ip地址和fd配置给作为参数传入的conn结构
        struct connection conn;
        connection_init(&conn, conf->tls ? server.ssl_ctx : NULL);
        int fd = accept_connection(&conn, serverfd);
        if (fd == 0)
            continue;
        if (fd < 0) {
            close_connection(&conn);
            break;
        }
        // 创建一个客户端结构, 用来持有conn和ev_ctx
        struct client *c = memorypool_alloc(server.pool);
        c->conn = conn;
        client_init(c);
        c->ctx = ctx;
        // 客户端添加到读取循环中
        ev_register_event(ctx, fd, EV_READ, read_callback, c);
        // 记录
        info.nclients++;
        info.nconnections++;
        log_info("[%p] Connection from %s", (void *) pthread_self(), conn.ip);
    }
}

// 读取数据包的回调, 每当客户端发来数据时由事件循环触发此函数
static void read_callback(struct ev_ctx *ctx, void *data) {
    // 客户端传入自身作为回调参数
    struct client *c = data;
    // 状态机校验, 也意味着只要是 WAITING_* 状态都需要继续读取数据
    if (c->status == SENDING_DATA)
        return;

    // 从客户端获取数据, 按照协议可了解数据是否已经读取完全
    int rc = read_data(c);
    switch (rc) {
        case 0:
            // 记录活跃时间
            c->last_seen = time(NULL);
            // 置为 SENDING 状态, 后续根据处理器的处理决定是否要发送数据
            c->status = SENDING_DATA;
            // 后续解码 + 处理器处理
            process_message(ctx, c);
            break;
        case -ERRCLIENTDC:
        case -ERRPACKETERR:
        case -ERRMAXREQSIZE:
            // 客户端断开或数据错误
            // 断开连接、清理资源
            log_error("Closing connection with %s (%s): %s",
                      c->client_id, c->conn.ip, solerr(rc));
            // 如果有遗嘱则发布遗嘱
            if (c->has_lwt == true) {
                char *tname = (char *) c->session->lwt_msg.publish.topic;
                struct topic *t = topic_get(&server, tname);
                publish_message(&c->session->lwt_msg, t);
            }
            // 清理资源
            ev_del_fd(ctx, c->conn.fd);
            // 从主题中删除订阅
            if (c->session && list_size(c->session->subscriptions) > 0) {
                struct list *subs = c->session->subscriptions;
                list_foreach(item, subs) {
                    log_debug("Deleting %s from topic %s",
                              c->client_id, ((struct topic *) item->data)->name);
                    topic_del_subscriber(item->data, c);
                }
            }
            client_deactivate(c);
            info.nclients--;
            info.nconnections--;
            break;
        case -ERREAGAIN:
            ev_fire_event(ctx, c->conn.fd, EV_READ, read_callback, c);
            break;
    }
}

// 此函数仅当客户端已经发送符合MQTT协议长度的完整字节流后才被调用
// 此函数使用事件循环基于收到的数据包类型做出反应, 在传入处理器前进行校验。
// 此函数根据处理器的输出结果, 在事件队列中加入回复事件, 或重置客户端继续监听输入事件
static void process_message(struct ev_ctx *ctx, struct client *c) {
    // io.data 是 mqtt_packet 类型
    struct io_event io = { .client = c };
    // 将收到的数据解码为mqtt包
    mqtt_unpack(c->rbuf + c->rpos, &io.data, *c->rbuf, c->read - c->rpos);
    // 重置读取标识
    c->toread = c->read = c->rpos = 0;
    // 使用对应的处理器处理
    c->rc = handle_command(io.data.header.bits.type, &io);
    switch (c->rc) {
        // 回复处理
        case REPLY:
        case MQTT_NOT_AUTHORIZED:
        case MQTT_BAD_USERNAME_OR_PASSWORD:
            // 向客户端发送数据
            enqueue_event_write(c);
            // 释放资源
            if (io.data.header.bits.type != PUBLISH)
                mqtt_packet_destroy(&io.data);
            break;
        // 断链处理
        case -ERRCLIENTDC:
            ev_del_fd(ctx, c->conn.fd);
            client_deactivate(io.client);
            info.nclients--;
            info.nconnections--;
            break;
        case -ERRNOMEM:
            log_error(solerr(c->rc));
            break;
        default:
            c->status = WAITING_HEADER;
            if (io.data.header.bits.type != PUBLISH)
                mqtt_packet_destroy(&io.data);
            break;
    }
}

// 写入事件触发的回调函数, 阻塞可重发, 发完后重置状态机, 并加入读取事件监听
static void write_callback(struct ev_ctx *ctx, void *arg) {
    struct client *client = arg;
    // 发送数据
    int err = write_data(client);
    switch (err) {
        case 0: // OK
            // 开启读取监听
            client->status = WAITING_HEADER;
            ev_fire_event(ctx, client->conn.fd, EV_READ, read_callback, client);
            break;
        // 阻塞重发
        case -ERREAGAIN:
            enqueue_event_write(client);
            break;
        default:
            log_info("Closing connection with %s (%s): %s %i",
                     client->client_id, client->conn.ip,
                     solerr(client->rc), err);
            ev_del_fd(ctx, client->conn.fd);
            client_deactivate(client);
            // Update stats
            info.nclients--;
            info.nconnections--;
            break;
    }
}
```

当然，启动的服务器必须进行阻塞调用以启动事件循环，我们也需要一个停止机制。得益于 ev_stop API，添加一个额外的事件例程来在我们想要停止运行的循环时调用变得非常简单。

现在我们的服务器会使用一个阻塞的循环来提供服务，但是我们也需要一个停止机制。感谢 `ev_stop` 接口，他这让我们可以简单的停止循环。

```c server.c
// 循环停止事件的回调函数, 由 EV_CLOSEFD 触发
static void stop_handler(struct ev_ctx *ctx, void *arg) {
    (void) arg;
    ev_stop(ctx);
}

// 事件循环启动函数, 是对 epoll 或者其他多路复用机制的抽象
static void eventloop_start(void *args) {
    int sfd = *((int *) args);
    struct ev_ctx ctx;
    ev_init(&ctx, EVENTLOOP_MAX_EVENTS);
    // 注册停止事件
    ev_register_event(&ctx, conf->run, EV_CLOSEFD|EV_READ, stop_handler, NULL);
    // 使用网络服务端口注册 accept_callback
    ev_register_event(&ctx, sfd, EV_READ, accept_callback, &sfd);
    // 注册周期性事件
    ev_register_cron(&ctx, publish_stats, NULL, conf->stats_pub_interval, 0);
    ev_register_cron(&ctx, inflight_msg_check, NULL, 0, 9e8);
    // 开始循环, 阻塞线程
    ev_run(&ctx);
    ev_destroy(&ctx);
}

// 添加一个写入事件监听, 用来向客户端发送数据
void enqueue_event_write(const struct client *c) {
    ev_fire_event(c->ctx, c->conn.fd, EV_WRITE, write_callback, (void *) c);
}
```

最终，我们的 `start_server` 函数，作为程序的入口，他会监听一个端口，并打开事件循环来提供服务。

```c server.c
// 服务入口, 传入地址和端口开始工作
int start_server(const char *addr, const char *port) {
    // Sol 全局对象初始化
    trie_init(&server.topics, NULL);
    server.authentications = NULL;
    server.pool = memorypool_new(BASE_CLIENTS_NUM, sizeof(struct client));
    server.clients_map = NULL;
    server.sessions = NULL;
    server.wildcards = list_new(wildcard_destructor);
    if (conf->allow_anonymous == false)
        config_read_passwd_file(conf->password_file, &server.authentications);
    // 服务器状态主题
    for (int i = 0; i < SYS_TOPICS; i++)
        topic_put(&server, topic_new(xstrdup(sys_topics[i].name)));
    // 监听网络端口
    int sfd = make_listen(addr, port, conf->socket_family);
    // 初始化SSL
    if (conf->tls == true) {
        openssl_init();
        server.ssl_ctx = create_ssl_context();
        load_certificates(server.ssl_ctx, conf->cafile, conf->certfile, conf->keyfile);
    }
    log_info("Server start");
    info.start_time = time(NULL);
    // 开启事件循环
    eventloop_start(&sfd);
    close(sfd);
    AUTH_DESTROY(server.authentications);
    list_destroy(server.wildcards, 1);
    // 释放SSL资源
    if (conf->tls == true) {
        SSL_CTX_free(server.ssl_ctx);
        openssl_cleanup();
    }
    log_info("Sol v%s exiting", VERSION);
    return 0;
}
```

正如你看到的，这里有一个用于创建客户端池的 `memorypool_new`，我们预先分配了一定数量的客户端，并且在断开链接时回收他们。只要我们的客户端内容是懒加载的，特别是他们的读写buffer（可能是 MB 级别）是懒加载的，那么我们这个客户端池就相当划算。

当然，这只是整个过程的一小部分，但最终我做出了一个相当不错的原型。下一步将是进行一些压力测试，看看它与 Mosquitto 或 Mosca 这些久经考验且无可争议的优秀软件相比如何。我们仍然缺少许多功能，例如用于存储会话的持久层，但先贼发布/订阅部分应该是可测试的。希望这个教程可以作为更整洁和精心设计的项目的起点。再见！

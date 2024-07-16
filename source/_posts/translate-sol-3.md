---
title: "[翻译]Sol - 从零开始的MQTT broker - 第三部分：服务"
url: translate-sol-3
date: 2023-12-22 09:44:08
categories:
- MQTT
tags:
- MQTT
- 翻译
- 网络编程
- 物联网
- C
---

> 原文 [Sol - An MQTT broker from scratch. Part 3 - Server](https://codepr.github.io/posts/sol-mqtt-broker-p3/)

<!--more-->

# 前言

这一部分我们会实现我们程序中的服务功能，通过之前在 [part-2](https://codepr.github.io/posts/sol-mqtt-broker-p2/) 中实现的 `network` 模块，我们可以比较轻松的接收并处理在 [part-1](https://codepr.github.io/posts/sol-mqtt-broker/) 中定义好的各种 `MQTT` 数据包。

# 服务端定义

我们的头文件非常简单，唯一向外提供的函数只有 `start_server`，他也只需要接收两个参数：

- 一个IP地址
- 一个监听端口

我们还需要定义创建 **epoll** 时使用的两个常量，一个是单次监听的最大事件数量，另一个是 **epoll** 监听的超时时间。这两个常量的定义以后我们也可以轻松的移动到配置模块里，暂时就先放在 `server` 的头文件。

```c src/server.h
// epoll 的默认配置
// 最大监听 256 事件
// -1 表示不超时, epoll 可以无限期的阻塞并监听
#define EPOLL_MAX_EVENTS    256
#define EPOLL_TIMEOUT       -1

// 不同类型的错误码
// client disconnection 客户端断开
// error reading packet 读包错误
// error packet sent exceeds size defined by configuration 包过大 (限制默认 2M)
#define ERRCLIENTDC         1
#define ERRPACKETERR        2
#define ERRMAXREQSIZE       3

// handler 的返回值, 表示对客户端读取后的下一个动作是读还是写
#define REARM_R             0
#define REARM_W             1

// 启动服务的函数
int start_server(const char *, const char *);
```

# 服务端实现

实现的部分比我一开始预想的要庞大一些，所有我们所需的 `处理器(handler)` 和 回调函数都会在这里定义。所以我们首先来实现三个最基础的回调函数，这三个函数是任何服务器都必不可少的：

- 用于建立连接的 `on_accept`
- 用于读取事件的 `on_read`
- 用于发送数据的 `on_write`

我们还需要定义一些关于MQTT包处理的 `handler`，同样使用一个数组保存，并且使 `handler` 在其中的序号等于包类型码。（这个方式我们已经用过好几次了）

```c src/server.c
#define _POSIX_C_SOURCE 200809L
#include <time.h>
#include <errno.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include "pack.h"
#include "util.h"
#include "mqtt.h"
#include "core.h"
#include "network.h"
#include "hashtable.h"
#include "config.h"
#include "server.h"

/* Seconds in a Sol, easter egg */
static const double SOL_SECONDS = 88775.24;

// 服务器本身状态信息
// 所有数据都会通过一个周期性回调发布
static struct sol_info info;

// broker 的全局实例, 包括了主题树和客户端的哈希表
static struct sol sol;

// 处理器接口
// 内含客户端的 closure 与数据包 mqtt_packet
typedef int handler(struct closure *, union mqtt_packet *);

// 包处理器, 每个函数负责处理对应名称的包
static int connect_handler(struct closure *, union mqtt_packet *);
static int disconnect_handler(struct closure *, union mqtt_packet *);
static int subscribe_handler(struct closure *, union mqtt_packet *);
static int unsubscribe_handler(struct closure *, union mqtt_packet *);
static int publish_handler(struct closure *, union mqtt_packet *);
static int puback_handler(struct closure *, union mqtt_packet *);
static int pubrec_handler(struct closure *, union mqtt_packet *);
static int pubrel_handler(struct closure *, union mqtt_packet *);
static int pubcomp_handler(struct closure *, union mqtt_packet *);
static int pingreq_handler(struct closure *, union mqtt_packet *);

// 处理器数组, 同样使用 type 的值作为索引
static handler *handlers[15] = {
    NULL,
    connect_handler,
    NULL,
    publish_handler,
    puback_handler,
    pubrec_handler,
    pubrel_handler,
    pubcomp_handler,
    subscribe_handler,
    NULL,
    unsubscribe_handler,
    NULL,
    pingreq_handler,
    NULL,
    disconnect_handler
};

// 本 module 内部使用的 conn 结构体, 用来接收新连接
struct connection {
    char ip[INET_ADDRSTRLEN + 1];
    int fd;
};

// I/O closures, 关于三个服务器主要操作的回调
// - 读取客户端发来的数据
// - 向客户端写数据
// - 接收新的客户端连接
static void on_read(struct evloop *, void *);
static void on_write(struct evloop *, void *);
static void on_accept(struct evloop *, void *);

// 定时回调, 周期性发布服务器状态
static void publish_stats(struct evloop *, void *);

// 从 sfd 接收一条新链接, 将他的 ip 和 fd 存入conn中
static int accept_new_client(int fd, struct connection *conn) {
    if (!conn)
        return -1;
    // 获得新链接
    int clientsock = accept_connection(fd);
    // 没有获取成功的话
    if (clientsock == -1)
        return -1;
    // 就是检查一些新连接的属性
    struct sockaddr_in addr;
    socklen_t addrlen = sizeof(addr);
    if (getpeername(clientsock, (struct sockaddr *) &addr, &addrlen) < 0)
        return -1;
    char ip_buff[INET_ADDRSTRLEN + 1];
    if (inet_ntop(AF_INET, &addr.sin_addr, ip_buff, sizeof(ip_buff)) == NULL)
        return -1;
    struct sockaddr_in sin;
    socklen_t sinlen = sizeof(sin);
    if (getsockname(fd, (struct sockaddr *) &sin, &sinlen) < 0)
        return -1;
    // 赋值我们要的 ip 和 fd
    conn->fd = clientsock;
    strcpy(conn->ip, ip_buff);
    return 0;
}

// accept 的回调, 通过 sfd 获得 cfd, 然后对 cfd 添加 EPOLLIN 监听
// loop evloop实例
// arg server closure, 包括了 sfd 在其中, on_accept 其实就是 server closure 的 call 参数
static void on_accept(struct evloop *loop, void *arg) {
    // arg 是 server closure
    struct closure *server = arg;
    struct connection conn;
    // 获得 conn
    accept_new_client(server->fd, &conn);
    // 创建这个客户端的 closure
    struct closure *client_closure = malloc(sizeof(*client_closure));
    if (!client_closure)
        return;
    // 填充内容
    client_closure->fd = conn.fd;
    client_closure->obj = NULL;                 // 闭包的主要对象, 这个项目中是 client 对象, 在第六部分定义
    client_closure->payload = NULL;
    client_closure->args = client_closure;      // 拿自己当回调参数
    client_closure->call = on_read;             // 数据来时触发 on_read
    generate_uuid(client_closure->closure_id);  // 生成uuid
    // 保存在一个哈希表里
    hashtable_put(sol.closures, client_closure->closure_id, client_closure);
    // 将这个 closure 注册到 evloop, 事件是 EPOLLIN
    evloop_add_callback(loop, client_closure);
    // 重置 server fd, 让其可以继续接收新链接
    evloop_rearm_callback_read(loop, server);
    // 记录新链接
    info.nclients++;
    info.nconnections++;
    // 日志
    sol_info("New connection from %s on port %s", conn.ip, conf->port);
}
```

正如你所见，我定义了两个静态函数（在C语言中，当我们不严格的追究术语时，由于这种静态函数只能被同样.c文件里的函数访问，我们可以把这种函数看作是其他OOP语言中的私有方法。）

`accept_new_client` 函数使用了上一篇文中 `network` 模块定义的 `accept_connection` 函数，得以从操作系统层级接收新连接并进行一些设置。`on_accept` 则是实际负责处理新链接的回调函数，他依赖 `accept_new_client` 函数。

`accept_new_client` 函数所需的参数结构 `connection` 是我从我其他项目的代码库复制过来的，并不是说必须要用这种方式。

```c src/server.c
// 接收数据流组装成数据包的函数, 被 on_read 回调使用
// 解析数据包头, 至少会包括 Fixed Header, 因为每个数据包都至少有 2byte 的 Fixed Header, 其中会包括包类型和剩余长度
// 入参包括
// clientfd 客户端fd
// buf 放置所有输入数据流
// command 表示mqtt包的第一个字节
static ssize_t recv_packet(int clientfd, unsigned char *buf, char *command) {
    // 总计读取的字节数
    ssize_t nbytes = 0;

    // 读取一个字节, 这里会包括 MQTT 类型字段
    if ((nbytes = recv_bytes(clientfd, buf, 1)) <= 0)
        return -ERRCLIENTDC;
    unsigned char byte = *buf;
    buf++;
    // 译者没有明白为何可以这样比较, 第一个byte应该是包括了 MQTT type 和 Flags 才对?
    if (DISCONNECT < byte || CONNECT > byte)
        return -ERRPACKETERR;

    // 逐字节读取变长的 Remaining Length
    unsigned char buff[4];
    int count = 0;
    int n = 0;
    do {
        // 使用 buf 读取
        if ((n = recv_bytes(clientfd, buf+count, 1)) <= 0)
            return -ERRCLIENTDC;
        // 并为 buff 赋值
        buff[count] = buf[count];
        nbytes += n;
        // 根据高位判断是否有后续
    } while (buff[count++] & (1 << 7));

    // 获得剩余长度的值
    const unsigned char *pbuf = &buff[0];
    unsigned long long tlen = mqtt_decode_length(&pbuf);

    // 判断是否过长
    if (tlen > conf->max_request_size) {
        nbytes = -ERRMAXREQSIZE;
        goto exit;
    }

    // 读取所有剩余的字节数, 获得完整数据包字节流
    // 译者认为这里 buf + 1 只考虑了 Remaining Length 长度为 1 的情况, 应改为 buf + count
    if ((n = recv_bytes(clientfd, buf + 1, tlen)) < 0)
        goto err;
    nbytes += n;
    // 第一个字节赋值为 command
    *command = byte;
exit:
    return nbytes;
err:
    shutdown(clientfd, 0);
    close(clientfd);
    return nbytes;
}

// 客户端输入数据的回调, 当 accepted 或 reply 之后等待
static void on_read(struct evloop *loop, void *arg) {
    // 这里带着一些客户端信息
    struct closure *cb = arg;

    // 使用最大数据包尺寸准备接收数据 默认2M
    unsigned char *buffer = malloc(conf->max_request_size);
    ssize_t bytes = 0;
    char command = 0;

    // 在此处必须完整的接收一个数据包的所有数据
    // 通过数据包的 Remaining Length 我们可以了解这个数据包的长度到底应该是多少
    bytes = recv_packet(cb->fd, buffer, &command);

    // 链接断开处理
    // TODO: 使用一个 error_handler 来处理 ERRMAXREQSIZE 将错误码返回给客户端
    if (bytes == -ERRCLIENTDC || bytes == -ERRMAXREQSIZE)
        goto exit;

    // 当我们收到一个错误的包时, 我们需要清理 buffer, 并断开这个客户端连接
    // 等客户端下次重连上来再处理
    if (bytes == -ERRPACKETERR)
        goto errdc;

    // 收包计数器
    info.bytes_recv++;

    // 将数据流解码为正确类型的mqtt包
    union mqtt_packet packet;
    unpack_mqtt_packet(buffer, &packet);
    union mqtt_header hdr = { .byte = command };

    // 然后找到对应的 hander 来处理这个包
    // 处理完的rc表示
    int rc = handlers[hdr.bits.type](cb, &packet);
    // 如果处理结果是需要发送一个包作为响应
    if (rc == REARM_W) {
        // 重置写入监听
        // 当 fd 可写入时 epoll 就会触发 EPOLLOUT 事件
        // cb 中的 call 会被执行, 也就是 on_write
        // 写入需要的参数, 会在 handlers 中会处理好, 之后由 cb 携带 
        cb->call = on_write;
        evloop_rearm_callback_write(loop, cb);
    } else if (rc == REARM_R) {
        // 重置读取监听, 后面有数据接着读
        cb->call = on_read;
        evloop_rearm_callback_read(loop, cb);
    }
    // Disconnect packet received
exit:
    free(buffer);
    return;
errdc:
    free(buffer);
    // 把客户端丢弃了
    sol_error("Dropping client");
    shutdown(cb->fd, 0);
    close(cb->fd);
    // 清理哈希表
    hashtable_del(sol.clients, ((struct sol_client *) cb->obj)->client_id);
    hashtable_del(sol.closures, cb->closure_id);
    // 记录信息
    info.nclients--;
    info.nconnections--;
    return;
}

// 写入回调, 当有需要写入的数据且 fd 可被写入时触发
static void on_write(struct evloop *loop, void *arg) {
    struct closure *cb = arg;
    ssize_t sent;
    // cb 里包括了所有需要发送的内容
    if ((sent = send_bytes(cb->fd, cb->payload->data, cb->payload->size)) < 0)
        sol_error("Error writing on socket to client %s: %s",
                  ((struct sol_client *) cb->obj)->client_id, strerror(errno));

    // 发包计数器
    info.bytes_sent += sent;
    // 释放
    bytestring_release(cb->payload);
    cb->payload = NULL;

    // 客户端的下一次触发肯定是 read (业务上来说服务端不可能连续发两个包)
    cb->call = on_read;
    evloop_rearm_callback_read(loop, cb);
}
```

我们又添加了三个静态函数，`recv_packet` 函数就像他的名字一样，依赖 `mqtt` 模块，负责持续接收数据流直到足够一个完整的 MQTT 包。另外两个分别是 `on_read` 和 `on_write`。

请注意，`on_read` 和 `on_write` 使用我们之前定义的函数不停的重置对 `socket` 的监听，就像来回打乒乓球一样。例如， `on_read` 可以通过 `处理器` 的返回值来决定下一次的操作是 `read` 还是 `write`，然后把客户端链接的下一个回调函数设置为 `on_read` 或者 `on_write`，当然也有可能是断开链接。比如说客户端发来的数据出现了错误，或者当客户端发来了 `DISCONNECT` 包，那么此时对应的 `处理器` 返回的值就既不是 `REARM_W` 也不是 `REARM_R`。

在 `on_write` 中我们看到 `send_bytes` 传入了一个带有大小和内容的 `payload`，这里使用了我定义的一个方便的工具结构 `bytestring`，我们现在就在 `src/pack.h` and `src/pack.c` 中添加他。

# 工具 bytestring

```c src/pack.h
// bytestring 结构体, 提供了一个便携的保存 bytes 的方法
// 他本质上提供了一个指向最后编辑位置的指针, 和 bytes 的总长度
struct bytestring {
    size_t size;
    size_t last;
    unsigned char *data;
};

// bytestring 的初始化函数, 需要一个长度作为参数
// 为了简化, 我们直接采用固定长度, 并且不会再后续使用过程中扩容
struct bytestring *bytestring_create(size_t);
void bytestring_init(struct bytestring *, size_t);
void bytestring_release(struct bytestring *);
void bytestring_reset(struct bytestring *);
```

这里是关于 `bytestring` 的实现。

```c src/pack.c
// 创建
struct bytestring *bytestring_create(size_t len) {
    struct bytestring *bstring = malloc(sizeof(*bstring));
    bytestring_init(bstring, len);
    return bstring;
}

// 初始化内部结构
void bytestring_init(struct bytestring *bstring, size_t size) {
    if (!bstring)
        return;
    bstring->size = size;
    bstring->data = malloc(sizeof(unsigned char) * size);
    bytestring_reset(bstring);
}

// 释放
void bytestring_release(struct bytestring *bstring) {
    if (!bstring)
        return;
    free(bstring->data);
    free(bstring);
}

// 清空数据
void bytestring_reset(struct bytestring *bstring) {
    if (!bstring)
        return;
    bstring->last = 0;
    memset(bstring->data, 0, bstring->size);
}
```

# 日志和通用工具

让我们稍微打断一下主线，按照我的经验，到这个阶段我们往往会需要一些工具函数，我一般会把他们统一放在 `util` 包中。我们刚才已经看到了一些 `sol_info`, `sol_debug` 或者 `sol_error` 这样的函数，其实就是 `util` 包中的定义。

我们的日志需求很简单，所以不需要专门做一个日志模块，就先放到 `util` 包里。

```c src/util.h
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <strings.h>

#define UUID_LEN     37  // 36 + nul char
#define MAX_LOG_SIZE 119

enum log_level { DEBUG, INFORMATION, WARNING, ERROR };

int number_len(size_t);
int parse_int(const char *);
int generate_uuid(char *);
char *remove_occur(char *, char) ;
char *append_string(char *, char *, size_t);

// 日志相关
void sol_log_init(const char *);
void sol_log_close(void);
void sol_log(int, const char *, ...);

#define log(...) sol_log( __VA_ARGS__ )
#define sol_debug(...) log(DEBUG, __VA_ARGS__)
#define sol_warning(...) log(WARNING, __VA_ARGS__)
#define sol_error(...) log(ERROR, __VA_ARGS__)
#define sol_info(...) log(INFORMATION, __VA_ARGS__)

#define STREQ(s1, s2, len) strncasecmp(s1, s2, len) == 0 ? true : false
```

log函数设置了一些宏定义，方便我们使用不同级别的日志。我们还做了一个 `STREQ` 用来比较两个字符串是否相等。

```c src/util.c
#include <time.h>
#include <ctype.h>
#include <errno.h>
#include <assert.h>
#include <string.h>
#include <stdlib.h>
#include <stdarg.h>
#include <uuid/uuid.h>
#include "util.h"
#include "config.h"

static FILE *fh = NULL;

// 通过文件保存日志
void sol_log_init(const char *file) {
    assert(file);
    fh = fopen(file, "a+");
    if (!fh)
        printf("%lu * WARNING: Unable to open file %s\n",
               (unsigned long) time(NULL), file);
}

void sol_log_close(void) {
    if (fh) {
        fflush(fh);
        fclose(fh);
    }
}

// 按级别写入内容
void sol_log(int level, const char *fmt, ...) {
    assert(fmt);
    va_list ap;
    char msg[MAX_LOG_SIZE + 4];
    if (level < conf->loglevel)
        return;
    va_start(ap, fmt);
    vsnprintf(msg, sizeof(msg), fmt, ap);
    va_end(ap);

    // 过长的信息会被截取, 然后加 ...
    memcpy(msg + MAX_LOG_SIZE, "...", 3);
    msg[MAX_LOG_SIZE + 3] = '\0';

    // Distinguish message level prefix
    const char *mark = "#i*!";

    // 同时写向标准输出和日志文件
    FILE *fp = stdout;
    if (!fp)
        return;
    fprintf(fp, "%lu %c %s\n", (unsigned long) time(NULL), mark[level], msg);
    if (fh)
        fprintf(fh, "%lu %c %s\n", (unsigned long) time(NULL), mark[level], msg);
    fflush(fp);
    if (fh)
        fflush(fh);
}

// 获得一个数字的字符串长度 如 number_len(321) => 3
int number_len(size_t number) {
    int len = 1;
    while (number) {
        len++;
        number /= 10;
    }
    return len;
}

// 解析字符串中的数字, 返回数字的值
int parse_int(const char *string) {
    int n = 0;
    while (*string && isdigit(*string)) {
        n = (n * 10) + (*string - '0');
        string++;
    }
    return n;
}

// 去除字符串中的某个字符
char *remove_occur(char *str, char c) {
    char *p = str;
    char *pp = str;
    while (*p) {          // 当 p 指向内容
        *pp = *p++;       // 1. 使用 *p 赋值 *pp 2. p右移 (保证每次原字符串读取下一个字符)
        pp += (*pp != c); // 仅当 *pp != c 时, pp 右移 (意味着如果时c则会被下一次写入覆盖)
    }
    *pp = '\0'; // pp的最新位置作为结尾
    return str;
}

// 将一个字符串添加到另一个字符串后面
// 前面是 src 后面是 chunk
char *append_string(char *src, char *chunk, size_t chunklen) {
    size_t srclen = strlen(src);
    char *ret = malloc(srclen + chunklen + 1);
    memcpy(ret, src, srclen);
    memcpy(ret + srclen, chunk, chunklen);
    ret[srclen + chunklen] = '\0';
    return ret;
}

// 创建 uuid
int generate_uuid(char *uuid_placeholder) {

    /* Generate random uuid */
    uuid_t binuuid;
    uuid_generate_random(binuuid);
    uuid_unparse(binuuid, uuid_placeholder);

    return 0;
}
```

这些简单的函数足以支撑我们的日志系统，如果在启动时调用 `sol_log_init` 我们还能将日志存入日志文件。

# 服务入口实现

终于我们要开始写 `start_server` 函数了，这个函数会调用所有我们之前写过的内容。他将作为程序的入口点，完成各种设置和全局实例的初始化，然后等待着客户端链接。

```c src/server.c
// 系统状态主题, 根据配置文件每 n 秒发布一次
#define SYS_TOPICS 14

static const char *sys_topics[SYS_TOPICS] = {
    "$SOL/",
    "$SOL/broker/",
    "$SOL/broker/clients/",
    "$SOL/broker/bytes/",
    "$SOL/broker/messages/",
    "$SOL/broker/uptime/",
    "$SOL/broker/uptime/sol",
    "$SOL/broker/clients/connected/",
    "$SOL/broker/clients/disconnected/",
    "$SOL/broker/bytes/sent/",
    "$SOL/broker/bytes/received/",
    "$SOL/broker/messages/sent/",
    "$SOL/broker/messages/received/",
    "$SOL/broker/memory/used"
};

// 一个阻塞的循环
static void run(struct evloop *loop) {
    if (evloop_wait(loop) < 0) {
        sol_error("Event loop exited unexpectedly: %s", strerror(loop->status));
        evloop_free(loop);
    }
}

// 在全局哈希表中删除客户端时触发回调释放资源
static int client_destructor(struct hashtable_entry *entry) {
    if (!entry)
        return -1;
    struct sol_client *client = entry->val;
    if (client->client_id)
        free(client->client_id);
    free(client);
    return 0;
}

// 在全局哈希表中删除闭包时触发回调释放资源
static int closure_destructor(struct hashtable_entry *entry) {
    if (!entry)
        return -1;
    struct closure *closure = entry->val;
    if (closure->payload)
        bytestring_release(closure->payload);
    free(closure);
    return 0;
}

// 启动服务器
int start_server(const char *addr, const char *port) {
    // 初始化 sol 全局实例
    trie_init(&sol.topics);
    // 确保所有的客户端和闭包都在哈希表中, 这样从哈希表删除时就可以使用回调释放资源
    sol.clients = hashtable_create(client_destructor);
    sol.closures = hashtable_create(closure_destructor);

    // 服务端 closure
    struct closure server_closure;
    // 开启端口监听
    server_closure.fd = make_listen(addr, port, conf->socket_family);
    server_closure.payload = NULL;
    server_closure.args = &server_closure;
    // 唯一事件是接受客户端链接
    server_closure.call = on_accept;
    generate_uuid(server_closure.closure_id);

    // 创建输出状态的基础 topic
    for (int i = 0; i < SYS_TOPICS; i++)
        sol_topic_put(&sol, topic_create(strdup(sys_topics[i])));
    // 创建 evloop
    struct evloop *event_loop = evloop_create(EPOLL_MAX_EVENTS, EPOLL_TIMEOUT);

    // 将服务端 closure 放入 evloop
    evloop_add_callback(event_loop, &server_closure);

    // 添加周期性事件 汇报服务器状态
    // TODO 实现
    struct closure sys_closure = {
        .fd = 0,
        .payload = NULL,
        .args = &sys_closure,
        .call = publish_stats
    };
    generate_uuid(sys_closure.closure_id);
    evloop_add_periodic_task(event_loop, conf->stats_pub_interval, 0, &sys_closure);
    // 初始化完成
    sol_info("Server start");
    info.start_time = time(NULL);
    // 进入事件循环
    run(event_loop);
    // 释放资源
    hashtable_release(sol.clients);
    hashtable_release(sol.closures);
    sol_info("Sol v%s exiting", VERSION);
    return 0;
}
```

# 定时通报服务器状态

好的，我们现在有了一个（几乎）功能齐全的服务器，它使用我们的回调系统来处理流量。 接下来我们需要在头文件上添加一些代码，例如我们刚才使用的 `info` 结构体，还有全局的名为 `sol` 的实例，这些我们都还没有定义。

```c src/server.h
// 全局 info
struct sol_info {
    int nclients;               // 当前客户端数
    int nconnections;           // 历史客户端总数
    long long start_time;       // 服务启动时间
    long long bytes_recv;       // 接收字节总数
    long long bytes_sent;       // 发送字节总数
    long long messages_sent;    // 发送消息总数
    long long messages_recv;    // 接收消息总数
};
```

这是刚才的 `start_server` 函数中我们添加的一个周期性任务。

```c
// 添加周期性事件 汇报服务器状态
// TODO 实现
struct closure sys_closure = {
    .fd = 0,
    .payload = NULL,
    .args = &sys_closure,
    .call = publish_stats
};
generate_uuid(sys_closure.closure_id);
evloop_add_periodic_task(event_loop, conf->stats_pub_interval, 0, &sys_closure);
```

`publish_stats` 函数会每隔 `conf->stats_pub_interval` 秒被调用一次， `conf->stats_pub_interval` 是一个全局的配置值，配置相关的内容我们稍后会去实现。

现在，让我们先实现这个回调函数：

```c src/server.c
// 发送消息的工具方法
static void publish_message(unsigned short pkt_id,
                            unsigned short topiclen,
                            const char *topic,
                            unsigned short payloadlen,
                            unsigned char *payload) {

    // 从全局的 topic 表中获得我们需发送的 topic, 如果不存在则退出
    struct topic *t = sol_topic_get(&sol, topic);
    if (!t)
        return;

    // 制作一个 PUBLISH 包
    union mqtt_packet pkt;
    struct mqtt_publish *p = mqtt_packet_publish(PUBLISH_BYTE,
                                                 pkt_id,
                                                 topiclen,
                                                 (unsigned char *) topic,
                                                 payloadlen,
                                                 payload);
    pkt.publish = *p;
    size_t len;
    unsigned char *packed;

    // 通过TCP向所有订阅了该主题的客户端发送 payload
    struct list_node *cur = t->subscribers->head;
    size_t sent = 0L;
    for (; cur; cur = cur->next) {
        sol_debug("Sending PUBLISH (d%i, q%u, r%i, m%u, %s, ... (%i bytes))",
                  pkt.publish.header.bits.dup,
                  pkt.publish.header.bits.qos,
                  pkt.publish.header.bits.retain,
                  pkt.publish.pkt_id,
                  pkt.publish.topic,
                  pkt.publish.payloadlen);
        len = MQTT_HEADER_LEN + sizeof(uint16_t) +
            pkt.publish.topiclen + pkt.publish.payloadlen;
        struct subscriber *sub = cur->data;
        struct sol_client *sc = sub->client;

        // 根据订阅者设置的 qos 更改包中的 qos
        pkt.publish.header.bits.qos = sub->qos;
        if (pkt.publish.header.bits.qos > AT_MOST_ONCE)
            len += sizeof(uint16_t);
        int remaininglen_offset = 0;
        if ((len - 1) > 0x200000)
            remaininglen_offset = 3;
        else if ((len - 1) > 0x4000)
            remaininglen_offset = 2;
        else if ((len - 1) > 0x80)
            remaininglen_offset = 1;
        len += remaininglen_offset;
        
        // 实际打包发送
        packed = pack_mqtt_packet(&pkt, PUBLISH);
        if ((sent = send_bytes(sc->fd, packed, len)) < 0)
            sol_error("Error publishing to %s: %s",
                      sc->client_id, strerror(errno));

        // 统计信息
        info.bytes_sent += sent;
        info.messages_sent++;
        free(packed);
    }
    free(p);
}

// 发送服务器状态的周期性任务
static void publish_stats(struct evloop *loop, void *args) {
    char cclients[number_len(info.nclients) + 1];
    sprintf(cclients, "%d", info.nclients);
    char bsent[number_len(info.bytes_sent) + 1];
    sprintf(bsent, "%lld", info.bytes_sent);
    char msent[number_len(info.messages_sent) + 1];
    sprintf(msent, "%lld", info.messages_sent);
    char mrecv[number_len(info.messages_recv) + 1];
    sprintf(mrecv, "%lld", info.messages_recv);
    long long uptime = time(NULL) - info.start_time;
    char utime[number_len(uptime) + 1];
    sprintf(utime, "%lld", uptime);
    double sol_uptime = (double)(time(NULL) - info.start_time) / SOL_SECONDS;
    char sutime[16];
    sprintf(sutime, "%.4f", sol_uptime);
    publish_message(0, strlen(sys_topics[5]), sys_topics[5],
                    strlen(utime), (unsigned char *) &utime);
    publish_message(0, strlen(sys_topics[6]), sys_topics[6],
                    strlen(sutime), (unsigned char *) &sutime);
    publish_message(0, strlen(sys_topics[7]), sys_topics[7],
                    strlen(cclients), (unsigned char *) &cclients);
    publish_message(0, strlen(sys_topics[9]), sys_topics[9],
                    strlen(bsent), (unsigned char *) &bsent);
    publish_message(0, strlen(sys_topics[11]), sys_topics[11],
                    strlen(msent), (unsigned char *) &msent);
    publish_message(0, strlen(sys_topics[12]), sys_topics[12],
                    strlen(mrecv), (unsigned char *) &mrecv);
}
```

我们已经注册了我们第一个周期性回调，他会定时的发送 `sys_topics` 数组中主题的消息，

下面是一些我们需要的全局实例：

```c src/server.c
// info 实例, 其内容会被周期性回调发送
static struct sol_info info;

// sol 实例, 包括 主题树 和 客户端哈希表
static struct sol sol;
```

# 结尾

我们还需要补充一些代码，才能使我们上面的代码能够运行。比如，`struct sol` 的定义、`closure_destructor` 函数，哈希表的定义，比如 `topic` 的存储和解析方法。这一切我们都需要去完成。

在下一部分我们会编写处理各种MQTT数据包的 `处理器`，根据数据包的类型和内容不同，服务器会表现出不同的行为。

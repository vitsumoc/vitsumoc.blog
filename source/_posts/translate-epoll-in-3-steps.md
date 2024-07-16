---
title: "[翻译]通过三个步骤简单理解epoll"
url: translate-epoll-in-3-steps
date: 2023-12-20 15:46:49
categories:
- 网络编程
tags:
- 网络编程
- 翻译
- C
---

> 原文 [epoll() Tutorial – epoll() In 3 Easy Steps!](https://suchprogramming.com/epoll-in-3-easy-steps/)

<!-- more -->

# 前言

就在不久前，能够让一台服务器[支持10000个并发连接](http://www.kegel.com/c10k.html)还是一个很了不起的事情。有很多因素让这个行为成为可能，例如 [nginx](https://www.nginx.com/)，他可以比他的前辈们更高效的处理更多连接。不过其中最大的因素应该还是大部分操作系统引入了恒定时间的轮询机制[O1](https://robbell.io/2009/06/a-beginners-guide-to-big-o-notation)，用来监视系统中的文件描述符。

在 [No Starch Press](https://nostarch.com/) 的书[《Linux 编程接口》](https://nostarch.com/tlpi)中，第 63.4.5 节提供了一个表格，描述了通过一些最常见的轮询方法检查不同数量的文件描述符所需的时间。

![](poll-times.png)

如图所示，在10个文件描述符时，epoll 已经体现出了他的性能优势。随着描述符数量的增加，相比于 [poll()](https://man7.org/linux/man-pages/man2/poll.2.html) 或 [select()](https://man7.org/linux/man-pages/man2/select.2.html)，这种性能优势体现的越来越大。

本教程将介绍在 Linux 2.6.27+ 上使用 epoll() 的一些基础知识。

## 预备知识

本教程假设您熟悉并熟悉 Linux、C 语法以及类 UNIX 系统中文件描述符的使用。

# 开始

创建一个新文件夹来开始我们的教程， Makefile 如下：

```makefile
all: epoll_example

epoll_example: epoll_example.c
	gcc -Wall -Werror -o $@ epoll_example.c

clean:
	@rm -v epoll_example
```

在这篇文章中，需要使用这些库：

```c epoll_example.c
#include <stdio.h>     // for fprintf()
#include <unistd.h>    // for close(), read()
#include <sys/epoll.h> // for epoll_create1(), epoll_ctl(), struct epoll_event
#include <string.h>    // for strncmp
```

# 第一步：创建 epoll 文件描述符

从最基础开始，先尝试创建和关闭 epoll 实例。

```c epoll_example.c
#include <stdio.h>     // for fprintf()
#include <unistd.h>    // for close()
#include <sys/epoll.h> // for epoll_create1()

int main()
{
	int epoll_fd = epoll_create1(0);
	
	if (epoll_fd == -1) {
		fprintf(stderr, "Failed to create epoll file descriptor\n");
		return 1;
	}

	if (close(epoll_fd)) {
		fprintf(stderr, "Failed to close epoll file descriptor\n");
		return 1;
	}

	return 0;
}
```

运行这段代码，正常来说应该直接返回并且不产生任何输出，如果你看到了错误消息，那么也许你可能正在运行一个非常旧的 Linux 内核。

第一个例子是使用 [epoll_create1()](https://linux.die.net/man/2/epoll_create1) 创建 `epoll` 实例，并且获得他的文件描述符。虽然我们没有用这个文件描述符做任何事情，我们仍然要记得在关闭程序之前清理他。就像和其他的 `Linux` 文件描述符一样，使用 `close()`。

## 电平触发（Level triggered）和边沿触发（edge triggered）

[电平触发和边沿触发](https://www.quora.com/What-are-the-key-differences-between-edge-triggered-and-level-triggered-interrupts) 是从电子工程师那边借来的术语，但当我们使用 `epoll` 时，我们需要注意这两者的差别。在边沿触发模式下，我们只会在被监控文件描述符的状态变化时接收到事件；而在电平触发模式下，我们会持续接收事件，直到被监控的文件描述符不再处于 ready 状态。一般来说电平触发时默认状态，而且更加容易上手，我们的教程也会使用电平触发。但是我们也需要直到有边沿触发这回事。

# 第二步：添加被 epoll 监控的文件描述符

接下来要做的事情就是，告诉 epoll 需要监控哪些文件描述符，以及需要监控哪种类型的事件。在这个例子里，我会使用Linux中我最爱的文件描述符，亲爱的 `file descriptor 0`（就是标准输入）。

```c epoll_example.c
#include <stdio.h>     // for fprintf()
#include <unistd.h>    // for close()
#include <sys/epoll.h> // for epoll_create1(), epoll_ctl(), struct epoll_event
	
int main()
{
	struct epoll_event event;
	int epoll_fd = epoll_create1(0);
	
	if (epoll_fd == -1) {
		fprintf(stderr, "Failed to create epoll file descriptor\n");
		return 1;
	}
	
	event.events = EPOLLIN;
	event.data.fd = 0;
	
	if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, 0, &event)) {
		fprintf(stderr, "Failed to add file descriptor to epoll\n");
		close(epoll_fd);
		return 1;
	}
	
	if (close(epoll_fd)) {
		fprintf(stderr, "Failed to close epoll file descriptor\n");
		return 1;
	}
	return 0;

}
```

这里我们创建了 `epoll_event` 的实例 `event`，并使用 [epoll_ctl()](https://linux.die.net/man/2/epoll_ctl) 将 `fd0` 添加到 epoll 的实例 `epoll_fd` 中。最后一个参数 `event` 是为了让 epoll 知道我们只想关注输入事件（`EPOLLIN`），而且还能为事件提供一些我们自定义的数据（本例中 `event.data.fd = 0`）。

# 第三步：完整例子

现在，让 epoll 发挥他的魔力吧

```c epoll_example.c
#define MAX_EVENTS 5
#define READ_SIZE 10
#include <stdio.h>     // for fprintf()
#include <unistd.h>    // for close(), read()
#include <sys/epoll.h> // for epoll_create1(), epoll_ctl(), struct epoll_event
#include <string.h>    // for strncmp

int main()
{
  // 是否运行中、当前并发事件数、计数器
	int running = 1, event_count, i;
  // 接收数据长度
	size_t bytes_read;
  // 接收输入 buffer
	char read_buffer[READ_SIZE + 1];
  // event 是一个事件结构 events 是事件数组, 最多5个
	struct epoll_event event, events[MAX_EVENTS];
  // epoll 实例
	int epoll_fd = epoll_create1(0);

	if (epoll_fd == -1) {
		fprintf(stderr, "Failed to create epoll file descriptor\n");
		return 1;
	}

  // 监听 EPOLLIN
	event.events = EPOLLIN;
	// 用户数据 fd = 0
  event.data.fd = 0;

  // 使用 epoll_ctl 添加监听
	if(epoll_ctl(epoll_fd, EPOLL_CTL_ADD, 0, &event))
	{
		fprintf(stderr, "Failed to add file descriptor to epoll\n");
		close(epoll_fd);
		return 1;
	}

	while (running) {
		// 等待输入
    printf("\nPolling for input...\n");
    // epoll_wait 等待事件发生
    // 返回值：接收并发事件数
    // 参数：epoll实例, 事件容器, 并发数, 超时时间
		event_count = epoll_wait(epoll_fd, events, MAX_EVENTS, 30 * 1000);
		printf("%d ready events\n", event_count);
		for (i = 0; i < event_count; i++) {
			printf("Reading file descriptor '%d' -- ", events[i].data.fd);
			bytes_read = read(events[i].data.fd, read_buffer, READ_SIZE);
			printf("%zd bytes read.\n", bytes_read);
			read_buffer[bytes_read] = '\0';
			printf("Read '%s'", read_buffer);
      // 输入为 stop 时结束
			if(!strncmp(read_buffer, "stop\n", 5))
			running = 0;
		}
	}

	if (close(epoll_fd)) {
		fprintf(stderr, "Failed to close epoll file descriptor\n");
		return 1;
	}

	return 0;
}
```

我们添加了一些变量，用来支撑这个例子，同时使用了一个循环，持续读取标准输入直到读取内容为 `stop`。我们使用 [epoll_wait()]() 来等待事件的发生，每个发生的事件都会被存储在 `events` 中，最大支持 `MAX_EVENTS` 个事件，并将超时事件设置为30秒。`epoll_wait()` 返回了本次触发了多少事件，然后我们只是在一个循环中打印这些事件而已。

# 使用实例

接下来是一些使用示例：

```text example
:~/epoll_example$ ./epoll_example

Polling for input...
hello
1 ready events
Reading file descriptor '0' -- 6 bytes read.
Read 'hello
'
Polling for input...
to looooooooooooong
1 ready events
Reading file descriptor '0' -- 10 bytes read.
Read 'to loooooo'
Polling for input...
1 ready events
Reading file descriptor '0' -- 10 bytes read.
Read 'ooooooong
'
Polling for input...
stop
1 ready events
Reading file descriptor '0' -- 5 bytes read.
Read 'stop
'
```

可以看到，第一次我们输入 `hello`，程序正确输出而且继续循环。

第二次当我们输入一个超过长度限制的输入 `to looooooooooooong` 时，电平触发机制帮助了我们。因为输入缓冲区一直有值，所以我们的事件就一直触发，直到读取完毕。在这种情况下，如果我们使用的是边沿触发，那么我们就只能收到一次通知，直到下次再有内容写入输入缓冲区时才会执行下一次事件了。

希望这篇文档能够帮助你使用 `epoll()`！
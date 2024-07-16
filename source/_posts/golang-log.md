---
title: Go标准库：log
url: Go-log
date: 2024-02-29 15:55:29
categories:
- Go
tags:
- Go
---

彪悍的人生不需要解释，那么彪悍的模块也不需要解释，例如今天所讨论的 `log` 模块，就绝对称得上是简明易懂、短小精干，只需稍微花个十来分钟阅读源码就可以理解并使用，非常具有 `go` 的味道。

<!-- more -->

# 示例

#### 最简单的应用：

```go test.go
func main() {
	log.Println("Hello Log!")
}
```

输出：

```log
2024/03/01 09:46:29 Hello Log!
```

#### 通过 Logger 对象指定写入位置和自定义前缀

```go test.go
func main() {
	file, _ := os.OpenFile("./test.log", os.O_RDWR|os.O_CREATE, 0666)
	defer file.Close()
	logger := log.New(file, "[Custom prefix]", log.LstdFlags)
	logger.Println("Hello Log!")
}
```

```log test.log
[Custom prefix]2024/03/01 09:51:53 Hello Log!
```

# 一个结构体

`log` 模块定义了名为 `Logger` 的结构体，用来打理一切和日志有关的操作：

```go log.go
type Logger struct {
	outMu sync.Mutex
	out   io.Writer // destination for output

	prefix    atomic.Pointer[string] // prefix on each line to identify the logger (but see Lmsgprefix)
	flag      atomic.Int32           // properties
	isDiscard atomic.Bool
}
```

`Logger` 中仅有5个成员变量，其含义如下：

| 成员 | 含义 |
| --- | --- |
| outMu | 输出锁，实现并发环境下有序的输出 |
| out | 输入对象，表示日志输出的目的地 |
| prefix | 用户设置的日志前缀 |
| flag | 配置项，一共有8个开关项 |
| isDiscard | 内部使用，记录输出对象是否为丢弃，当输出目标是丢弃时，输出函数不再处理，直接返回 |

`flag` 表示的配置项有这些：

| 配置项 | 含义 | 示例 |
| --- | --- | --- |
| Ldate | 添加日期 | 2009/01/23 |
| Ltime | 添加时间 | 01:23:23 |
| Lmicroseconds | 添加微秒 | 01:23:23.123123 |
| Llongfile | 添加完整的日志产生的文件和行数 | /a/b/c/d.go:23 |
| Lshortfile | 添加简短的日志产生的文件和行数 | d.go:23 |
| LUTC | 使用UTC时间，不使用本地时区 |  |
| Lmsgprefix | 将前缀至于行首，而非消息前 | 2024/03/01 10:30:58 [Custom prefix]Hello Log! |
| LstdFlags | 等于 Ldate \| Ltime | 设置日期时间的快捷方法 |

# 两种入口

使用者可以自己创建 `Logger` 对象，从而定制日志输出的目标、字符串前缀、配置项等，然而更简单的方式是直接使用 `log` 模块提供的公共函数。这些函数使用 `log` 模块内置的 `Logger` 对象、使用 `os.stdOut` 作为输出、没有前缀并使用 `LstdFlags` 作为默认配置。

```go
  log.Println("Hello Log!")
```

或者自己创建 `Logger` 实现更多的定制化需求，或是创建多个 `Logger` 将日志发往不同地方等：

```go
logger := log.New(file, "[Custom prefix]", log.LstdFlags)
```

# 三类输出方式

不管是如何创建的 `Logger` ，都提供了相同的输出方式，即三类、六种功能函数：

- Print[f|ln]
- Fatal[f|ln]
- Panic[f|ln]

| 函数 | 含义 |
| --- | --- |
| Printf | 输出日志 |
| Println | 输出日志并换行 |
| Fatalf | 输出日志，之后退出程序 |
| Fatalln | 输出日志并换行，之后退出程序 |
| Panicf | 输出日志，之后向上抛出panic，可能退出程序或被recover |
| Panicln | 输出日志并换行，之后向上抛出panic，可能退出程序或被recover |

# 总结

`Go` 提供的日志模块非常简单，清晰易懂，便于使用者将其封装为各种业务模块。使用方法简单到堪称优雅的程度，再次提醒我们在使用 `Go` 工作的过程中，要反复不断的去学习标准库，体会其少即是多的设计思想。
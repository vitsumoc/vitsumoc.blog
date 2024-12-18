---
title: "Go项目日志实践"
url: "Go项目日志实践"
date: 2024-12-18 09:43:51
categories:
- Go
tags:
- Go
- 项目实践
---

伴随着项目的成长，代码的规模越来越大、组件的关系越来越繁杂、部署的点位也越来越多，人肉运维逐渐难以维护项目的运行，日志的重要性的越发体现出来。

<!-- more -->

这次的改造涉及到项目中的多个 Go 程序，需要实现下列功能：

- 将程序运行时的信息写入日志文件
- 将程序崩溃时的错误写入日志文件
- 日志文件可以滚动更新

为了实现上述功能，使用了一些标准库和开源库，分别是：

- runtime: 用来获得 panic 信息
- [lumberjack](https://github.com/natefinch/lumberjack): 实现了滚动日志文件写入

# 代码实现

有了上述这些库的帮助，实际上我要做的事情已经很少了，大概还有这些：

- 配置开源库，文件大小、保存位置、保存数量等等
- 实现日志分级，在需要的时候开启 DEBUG，平时就输出 INFO 和 ERROR
- gin 框架 recovery 时写入日志，主程序 panic 时写入日志

最终的实现是这样：

```go runlog.go
package vlog

import (
	"fmt"
	"net/http"
	"runtime/debug"
	"time"

	"github.com/gin-gonic/gin"
	"gopkg.in/natefinch/lumberjack.v2"
)

var RunLogger *lumberjack.Logger
var VRunLogLevel RunLogLevel = RL_INFO

// 运行日志初始化
func InitRunLog() {
	RunLogger = &lumberjack.Logger{
		Filename:   "./logs/col_run.log",
		MaxSize:    50,    // mb
		MaxBackups: 20,    // 最大文件保留
		MaxAge:     30,    // 最大天数保留
		LocalTime:  false, // 使用UTC时间
		Compress:   false, // 不压缩
	}
	RunLog(RL_INFO, "运行日志初始化...")
}

// 日志等级
type RunLogLevel int

const (
	RL_DEBUG RunLogLevel = iota
	RL_INFO
	RL_ERROR
)

// 写入日志
func RunLog(level RunLogLevel, format string, a ...any) {
	if level < VRunLogLevel {
		return
	}
	// 获取当前时间
	now := time.Now()
	timeStr := fmt.Sprintf("[%d-%02d-%02d %02d:%02d:%02d.%03d] ",
		now.Year(), now.Month(), now.Day(),
		now.Hour(), now.Minute(), now.Second(),
		now.Nanosecond()/1000000)
	// 日志等级
	var levelStr string
	switch level {
	case RL_DEBUG:
		levelStr = "[DEBUG] "
	case RL_INFO:
		levelStr = "[INFO] "
	case RL_ERROR:
		levelStr = "[ERROR] "
	}
	// 写入日志
	RunLogger.Write([]byte(timeStr + levelStr + fmt.Sprintf(format, a...) + "\n"))
	fmt.Println(timeStr + levelStr + fmt.Sprintf(format, a...))
}

// WEB恢复
func WebRecovery() gin.HandlerFunc {
	return func(c *gin.Context) {
		defer func() {
			if err := recover(); err != nil {
				// 记录错误信息，可以使用自己喜欢的日志库
				RunLog(RL_ERROR, "发生panic:\n%v", err)
				stack := debug.Stack()
				RunLog(RL_ERROR, "调用栈信息:\n%s", stack)
				// 设置响应状态码为500
				c.JSON(http.StatusInternalServerError, gin.H{"error": "内部服务器错误"})
			}
		}()
		// 继续执行下一个中间件或路由处理函数
		c.Next()
	}
}

// Panic 记录
func LogPanic() {
	stack := debug.Stack()
	RunLog(RL_ERROR, "主程序异常退出")
	RunLog(RL_ERROR, "调用栈信息:\n%s", stack)
}

```

DEBUG 等级的日志可以在程序运行时通过 flag 开启，如下：

```go main.go
debug := flag.Bool("debug", false, "是否开启调试模式")
flag.Parse()
if *debug {
	vlog.VRunLogLevel = vlog.RL_DEBUG
}

```

`WebRecovery` 是写给 `gin` 框架的回调，用来在 WEB 过程中 panic 时记录日志，用法如下：

```go web.go
r := gin.Default()
r.Use(vlog.WebRecovery(), gin.LoggerWithConfig(gin.LoggerConfig{
	Output: vlog.RunLogger,
}))

```

最后，如果程序本身崩溃了，那么也需要记录到日志中：

```go main.go
func main() {
  defer vlog.LogPanic()
  // 其他初始化
}

```
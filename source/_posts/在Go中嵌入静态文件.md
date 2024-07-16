---
title: 在Go中嵌入静态文件
url: Go-embed
date: 2024-04-25 15:48:11
categories:
- Go
tags:
- Go
---

> 本文内容来自于 [embed](https://pkg.go.dev/embed)

<!-- more -->

# 概述

embed 包提供了在 Go 程序中嵌入静态文件以及访问 Go 程序中静态文件的能力。

Go 程序可以通过引入 embed 包并使用 //go:embed 指令在编译期将类型为 string、[]byte 或 FS 的变量初始化为文件或文件夹的内容。

例如，这里提供了三种嵌入 hello.txt 文件并打印内容的示例。

将单个文件嵌入为 string：

```go
import _ "embed"

//go:embed hello.txt
var s string
print(s)
```

将单个文件嵌入为 bytes 切片：

```go
import _ "embed"

//go:embed hello.txt
var b []byte
print(string(b))
```

将一个或多个文件嵌入为文件系统（FS）：

```go
import "embed"

//go:embed hello.txt
var f embed.FS
data, _ := f.ReadFile("hello.txt")
print(string(data))
```

## 指令

变量声明上方的 //go:embed 指令使用一条或多条 path.Match 模板指定要嵌入的文件。

指令必须直接位于单个变量声明的上方，在指令和变量声明之间只允许存在空行或 // 开头的行注释。

变量的类型必须是 string、byte 切片或者 FS（或者 FS 的别名）。

例如：

```go
package server

import "embed"

// content holds our static web server content.
//go:embed image/* template/*
//go:embed html/index.html
var content embed.FS
```

Go 构建系统将识别指令并安排声明的变量（在上面的示例中为 content）用文件系统中的匹配文件填充。

为了简洁起见，//go:embed 指令允许在一行中使用多个空格分隔的模板，同时为了避免当模板太多时造成过长的行，指令也允许被分为多行。模板使用源文件包所在的相对路径解析，使用 / 分隔路径，即便在 Windows 系统中也是如此。模板不能包含 '.' '..' 或空路径元素，也不能以 / 开始或结尾。如果需要匹配当前路径中的所有元素，请使用 '*' 而不是 '.'。为了允许支持名称中包含空格的文件，可以将模板写为 Go 格式的双引号或反引号字符串。

如果模板内容是文件夹名称，则文件夹中所有的内容都会被嵌入（递归），除了以 '.' 或 '_' 开头的文件。因此上述例子中的变量声明等同于：

```go
// content is our static web server content.
//go:embed image template html/index.html
var content embed.FS
```

区别在于 'image/*' 嵌入了 'image/.tempfile' 而 'image' 不会嵌入，两种写法都不会嵌入 'image/dir/.tempfile'。

如果模板使用 'all:' 前缀开头，则在遍历时会加入 '.' 和 '_' 开始的文件，例如，'all:image' 既可以嵌入 'image/.tempfile' 也可以嵌入 'image/dir/.tempfile'。

//go:embed 指令既可以用于导出的或不导出的变量，这取决于包是否想将数据内容提供给其他包使用。指令只能被用于包层级的变量，不能被用于本地变量。

模板不能用来匹配包之外的文件，例如 '.git/*' 或是应用链接，模板不能用来匹配名称带有特殊字符的文件，<code>" * < > ? ` ' | / \ :</code>，对空文件夹的匹配会被忽略，此外，每个 //go:embed 行中的模板至少需要匹配到一个文件或者一个非空的文件夹。

如果模板无效或匹配内容无效，构建将会失败。

## Strings 和 Bytes

匹配 string 或 []byte 类型变量的 //go:embed 指令必须只包含一个模板，而且模板必须只匹配到一个文件。string 或 []byte 变量的值将被初始化为文件的内容。

使用 //go:embed 指令需要先引入 embed 包，即便是仅使用 string 或 []byte，不使用 embed.FS。此时可以使用空标识符引入（import _ "embed"）。

## File Systems

对于单个文件的嵌入，最佳方式往往是 string 或 []byte。FS 类型允许嵌入一组树状结构的文件，例如一个静态 WEB 服务器的内容，就像上文中例子里提到过的那样。

FS 实现了 io/fs 包中的 FS 接口，因此它可以被任何理解该接口的包使用，例如 net/http，text/template 和 html/template。

例如，通过使用上文例子中的 content 变量，我们可以：

```go
http.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.FS(content))))

template.ParseFS(content, "*.tmpl")
```

## 工具

为了支持 Go 包分析工具，//go:embed 指令中的模板也会在 "go list" 输出中可见，请查看 "go help list" 中的 EmbedPatterns，TestEmbedPatterns 和 XTestEmbedPatterns 部分。

## 完整示例

```go
package main

import (
	"embed"
	"log"
	"net/http"
)

//go:embed internal/embedtest/testdata/*.txt
var content embed.FS

func main() {
	mux := http.NewServeMux()
	mux.Handle("/", http.FileServer(http.FS(content)))
	err := http.ListenAndServe(":8080", mux)
	if err != nil {
		log.Fatal(err)
	}
}
```
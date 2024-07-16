---
title: "[翻译]Effective Go"
url: effective_go
date: 2024-03-25 15:22:03
categories:
- Go
tags:
- Go
- 翻译
---

> 原文 [Effective Go](https://go.dev/doc/effective_go)

<!-- more -->

目录

- [简介](#简介)
  - [示例](#示例)
- [格式](#格式)
- [注释](#注释)
- [命名](#命名)
  - [包命名](#包命名)
  - [Getters](#Getters)
  - [接口命名](#接口命名)
  - [驼峰命名](#驼峰命名)
- [分号](#分号)
- [控制结构](#控制结构)
  - [If](#If)
  - [重声明与重赋值](#重声明与重赋值)
  - [For](#For)
  - [Switch](#Switch)
  - [Type switch](#Type-switch)
- [函数](#函数)
  - [多返回值](#多返回值)
  - [命名结果参数](#命名结果参数)
  - [Defer](#Defer)
- [数据](#数据)
  - [通过new分配](#通过new分配)
  - [构造器与复合字面量](#构造器与复合字面量)
  - [通过make分配](#通过make分配)
  - [数组](#数组)
  - [切片](#切片)
  - [二维切片](#二维切片)
  - [Maps](#Maps)
  - [打印](#打印)
  - [Append](#Append)
- [初始化](#初始化)
  - [常量](#常量)
  - [变量](#变量)
  - [init函数](#init函数)
- [方法](#方法)
  - [指针或是值](#指针或是值)
- [接口和其他类型](#接口和其他类型)
  - [接口](#接口)
  - [转换](#转换)
  - [接口转换与类型断言](#接口转换与类型断言)
  - [概论](#概论)
  - [接口与方法](#接口与方法)
- [空标识符](#空标识符)
  - [多重赋值中的空标识符](#多重赋值中的空标识符)
  - [未使用的引入和变量](#未使用的引入和变量)
  - [为了潜在作用而引入](#为了潜在作用而引入)
  - [接口检查](#接口检查)
- [嵌入式](#嵌入式)
- [并发](#并发)
  - [通过通信共享](#通过通信共享)
  - [Goroutines](#Goroutines)
  - [Channels](#Channels)
  - [Channels of channels](#Channels-of-channels)
  - [并行处理](#并行处理)
  - [A leaky buffer](#A-leaky-buffer)
- [错误](#错误)
  - [Panic](#Panic)
  - [Recover](#Recover)
- [一个WEB服务器](#一个WEB服务器)

<head>
  <style>
    .indentation {
      margin-left: 2rem;
    }
  </style>
</head>

# 简介

Go 是一门新语言。虽然他借鉴了现有编程语言的思想，但他也具有独特的特性，使得有效的 Go 程序与其他语言编写的程序有性质上的不同。直接将 C++ 或 Java 程序改编为 Go 程序可能无法得到一个让人满意的结果——Java 程序是使用 Java 编写的，而不是 Go。另一方面，从 Go 的角度思考问题可能会产生一个成功但完全不同的程序。换句话说，要写好Go，重要的是要理解它的特性和习惯用法。了解 Go 编程的既定约定也很重要，例如命名、格式、程序构造等，这样您编写的程序将易于其他 Go 程序员理解。

这份文档提供了编写清晰、地道的 Go 代码的建议。他补充了 [language specification](https://go.dev/ref/spec)、[the Tour of Go](https://go.dev/tour/)以及 [How to Write Go Code](https://go.dev/doc/code.html) 这几份资料，建议您在阅读本指南之前先阅读这些资料。

2022 年 1 月注：这份指南最初发布于 2009 年，自那以来更新并不频繁。虽然他仍然是理解如何使用 Go 语言本身的良好指南（因为语言本身很稳定），但他几乎没有涉及到 Go 生态系统自发布以来的重大变化，例如构建系统、测试、模块化和多态性。由于近年来变化太多，官方也不会再更新此文档。目前已经有很多文档、博客和书籍详细描述了现代 Go 的用法，足以满足学习需求。因此，虽然这份「Effective Go」指南仍然可用，但读者需要理解它并不是全面的指南。详情请参阅 [issue 28782](https://go.dev/issue/28782)。

## 示例

[Go package sources](https://go.dev/src/)不仅是核心库，还包含了如何使用该语言的示例。此外，许多标准库包都包含可直接在 [go.dev](https://go.dev/) 网站上运行的独立可执行示例（例如[本例](https://go.dev/pkg/strings/#example-Map)，如果需要，可以点击 Example 按钮打开它）。如果您对如何解决问题或如何实现某项功能有疑问，那么标准库中的文档、代码和示例可以提供答案、思路和背景信息。

# 格式

格式问题是争议最大但影响最小的问题。人们可以适应不同的格式风格，但如果没有必要的话，最好保持统一，这样大家就可以少花时间在格式上争论。 难点在于如何在没有冗长强制性风格指南的情况下实现这种「乌托邦」式的统一格式。

对于 Go 代码格式，我们采用了一种不同寻常的方法，让机器来处理大多数格式化问题。gofmt 程序（也可以称为 go fmt，它在包级别而不是源文件级别工作）读取一个 Go 程序，并按照标准的缩进和垂直对齐样式输出源代码，同时保留并根据需要重新格式化注释。如果您想了解如何处理一些新的布局情况，请运行 gofmt； 如果答案看起来不对，请重新排列您的程序（或提交有关 gofmt 的 bug 报告），不要试图绕过它。

例如，不必浪费时间去对齐结构体中的注释，Gofmt 会协助你处理，对于如下声明：

```go
type T struct {
    name string // name of the object
    value int // its value
}
```

gofmt 会将列对其：

```go
type T struct {
    name    string // name of the object
    value   int    // its value
}
```

所有标准库中的 Go 代码都使用 gofmt 进行了格式化。

简要的介绍一下格式化的细节：

缩进

<div class="indentation">
我们使用制表符 (tab) 进行缩进，gofmt 工具默认也会采用这种方式。只有在绝对必要的情况下才使用空格。
</div>

<br />

行长

<div class="indentation">
Go 没有代码行长限制。不用担心代码会超出打孔卡的限制。如果一行代码看起来太长，可以将其换行并使用额外的制表符缩进。
</div>

<br />

括号

<div class="indentation">
与 C 和 Java 语言相比，Go 需要更少的括号：控制结构 (if、for、switch) 的语法本身不需要括号。此外，Go 的运算符优先级层次更短更清晰。因此

```go
x<<8 + y<<16
```

通过间隔暗示了执行顺序，这一点和其他编程语言不同。
</div>

# 注释

Go 提供了 C 风格的 /* */ 块状注释和 C++ 风格的 // 行注释。通常情况下会使用行注释，块状注释往往是在包注释中使用，但也可以用在表达式中或禁用大块的代码。

位于顶层声明之前的注释，如果没有额外的空行隔开，就会被视为该声明的文档。这些“文档注释”是 Go 程序包或命令的主要文档形式。有关文档注释的更多信息，请参阅“[Go Doc Comments](https://go.dev/doc/comment)”。

# 命名

在 Go 中，命名和其他语言一样重要。他们甚至具有语义上的影响：一个名称对于包外部的可见性取决于他的首字母是否大写。因此，值得花一点时间讨论 Go 程序中的命名约定。

## 包命名

当一个包被引入时，包名称就成为了其内容的入口，当做了如下引入之后

```go
import "bytes"
```

引入该包的程序可以使用 bytes.Buffer。所有人都使用相同的名称来引用包的内容有很多好处，这也意味着包的命名必须是优秀的：短，简洁，意义明确。按照约定，包使用小写单个单词命名，不应出现下划线或驼峰命名。宁可过于简洁，因为每个使用您的包的人都需要输入这个名称。也无需担心包名的重复，包名只是引入时的默认名称，他不必在所有的源码中保持唯一，在极少数的包名重复的情况下，引入包可以选择一个本地使用的别名。无论如何，包名重复都是很罕见的，因为引入中的文件名决定了在使用哪个包。

另一个约定是包名是他源文件夹的基础名称，src/encoding/base64 中的包被 "encoding/base64" 引入，但使用名称为 base64，并非 encoding_base64 或 encodingBase64。

引入包的程序会使用包名来引用包中的内容，因此包导出的名称可以利用这一点来避免重复（不要使用 import . 来省略包名，他可以简单的运行一些必须在包外部运行的测试，但应该在其他场合下避免）。例如，在 bufio 包中的缓冲读取器被命名为 Reader，而不是 BufReader，因为使用者看到的将会是 bufio.Reader，这是一个清晰、简洁的名称。此外，由于引入的内容总是和引入的包名一起使用，因此 bufio.Reader 并不会和 io.Reader 冲突。相似的，创建 ring.Ring 新实例的函数——也就是 Go 中的构造函数——通常被命名为 NewRing，但是考虑到 Ring 是 ring 包中导出的唯一类型，而且又因为包名已经是 ring，所以这个函数只需被命名为 New，包的使用者看到的是 ring.New。利用包的结构可以帮助你选择合适的命名。

另一个简短的示例是 once.Do，once.Do(setup) 读起来非常友好，而且明显优于 once.DoOrWaitUntilDone(setup)。长的命名不一定能带来更丰富的含义。一个实用的文档注释常常比长的命名更有价值。

## Getters

Go 没有提供对 getters 和 setters 的自动支持。但您可以自己添加 getters 和 setters，而且这也常常是很适合的，但是在 getter 的名称前加上 Get 即不必须，也不符合 Go 的习惯。假设你有一个名为 owner 的字段（小写，非导出），那么他的 getter 方法应该被命名为 Owner（大写，导出），而不是 GetOwner。使用大写名称来作为导出标志可以方便的区分字段和方法。如果需要的话，他的 setter 函数应该被命名为 SetOwner。这些名称在实践中具有很好的阅读性：

```go
owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}
```

## 接口命名

按照约定，只有一个方法的接口会使用方法名加 -er 后缀或类似的修改，构成一个表示操作者的名词：Reader，Writer，Formatter，CloseNotifier 等等。

许多常用方法名已经约定俗成，遵循这些既有命名并保持其含义是高效的。Read，Write，Close，Flush，String 等都拥有标准的方法签名和含义。为了避免混淆，不要给你的方法起这些名称，除非它们的方法签名和含义完全相同。相反地，如果你的实现具有和现有类型完全相同的方法签名和含义，那么就应该使用相同的名称和方法签名。例如，将你的字符串转换方法命名为 String，而不是 ToString。

## 驼峰命名

最后，Go 中处理多个单词的名称时使用大驼峰命名或小驼峰命名，而非下划线连接。

# 分号

和 C 相同，Go 的语法使用分号来终止语句，但与 C 不同的是，这些分号不会出现在源码中。取而代之的是，词法分析器会使用一个简单的规则在扫描时自动插入分号，因此输入的文本基本上无需含有分号。

规则是这样的。如果在换行符之前的最后一个词是类型标志（比如 int 和 float64），基本字面量（比如一个数字或字符串），或是以下之一：

```text
break continue fallthrough return ++ -- ) }
```

词法分析器会在这些词后插入一个分号。可以这样总结“如果一个换行符前的词可以使用分号结尾，则插入一个分号”。

右大括号前的分号也可以省略，因此这样的语句

```go
 go func() { for { dst <- <-src } }()
```

无需分号。地道的 Go 程序只会在类似于 for 循环处使用分号，用来区分初始化语句、约束条件和循环动作。如果您必须要将多条语句写入同一行中，那么他们之间也需要分号。

分号插入规则带来的后果之一是你不能将左大括号放置在控制语句（if，for，switch，select）的下一行。如果您这样做了，大括号前会被插入一个分号，这会带来一些不良影响。应该这样写

```go
if i < f() {
    g()
}
```

而不是这样写

```go
if i < f()  // wrong!
{           // wrong!
    g()
}
```

# 控制结构

Go 中的控制结构与 C 息息相关，但在一些重要的方面又有所不同。没有 do 或 while 循环，只有一个更通用的 for；switch 更加灵活；if 和 switch 像 for 一样可以设置初始化语句；break 和 continue 语句采用可选标签来标识需要中断或继续的内容；有一些新的控制结构例如type switch 和一个多路选择器 select。语法也略有不同：没有小括号，并且主体内容必须用大括号包裹。

## If

Go 中简单的 If 看起来是这样：

```go
if x > 0 {
    return y
}
```

强制要求的大括号鼓励使用多行书写简单的 if 语句。无论如何，这都是一种好的代码风格，特别是当主体内容包含 return 或 break 之类的控制语句的时候。

由于 if 和 switch 可以接收一个初始化语句，因此通常会看到他被用来设置局部变量。

```go
if err := file.Chmod(0664); err != nil {
    log.Print(err)
    return err
}
```

在 Go 库中，你会发现当 if 语句不流入下一个语句时——即内容主体使用 break，continue，goto 或 return 结尾——不必要的 else 会被省略。

```go
f, err := os.Open(name)
if err != nil {
    return err
}
codeUsing(f)
```

这是一个常见情况的示例，其中的代码必须考虑一系列的错误情况。代码的可读性很好，成功的步骤按序向下执行，错误总是在他出现的地方被处理。由于错误情况都使用 return 语句结束流程，因此不需要使用 else 语句。

```go
f, err := os.Open(name)
if err != nil {
    return err
}
d, err := f.Stat()
if err != nil {
    f.Close()
    return err
}
codeUsing(f, d)
```

## 重声明与重赋值

上一节的例子通过调用 os.Open 函数展示了如何使用 := 进行短声明

```go
f, err := os.Open(name)
```

这行代码声明了两个变量，f 和 err。几行后，又调用了 f.Stat

```go
d, err := f.Stat()
```

这看起来是声明了 d 和 err。但请注意，这两行代码都出现了 err。这种重复是合法的：err 只被第一行代码声明，在第二行代码中只是被*重赋值（re-assigned）*。这意味着调用 f.Stat 使用的是之前声明的 err，此处仅仅是给他赋予新的值。

在一个 := 赋值中，已经被声明的变量 v 仍然可以被重赋值，只要：

- 本次声明和已经存在的对 v 的声明在同一个代码空间（如果 v 是在外部空间被声明的，那么本次声明会创造一个新的变量§），
- 初始化中相应的值可以被赋值给 v，而且
- 本次声明中至少包括一个全新被声明的变量。

这样的非常规设计是纯粹的实用主义，使得使用单个 err 变得很容易，例如在一长串的 if-else 中。你会经常看到这样的用法。

§ 这里值得注意的是，在 Go 中，函数的参数和返回值的空间与函数体相同，即使它们在词法上出现在函数体的大括号之外。

## For

Go 中的 for 循环和 C 中的 for 循环相似但又不同。他统一了 for 循环和 while 循环，并且没有 do-while 循环。总共有三种形式，其中只有一种带有分号。

```go
// Like a C for
for init; condition; post { }

// Like a C while
for condition { }

// Like a C for(;;)
for { }
```

短声明方式让声明循环中使用的索引变得容易。

```go
sum := 0
for i := 0; i < 10; i++ {
    sum += i
}
```

如果您在循环数组、切片、字符串或 map，又或是在读取 channel，range 关键字可以帮助你管理循环。

```go
for key, value := range oldMap {
    newMap[key] = value
}
```

如果您只需要 range 中的第一个参数（key 或者 index），那么丢掉第二个：

```go
for key := range m {
    if key.expired() {
        delete(m, key)
    }
}
```

如果你只需要 range 中的第二个参数（值），使用空标识，也就是下划线，从而丢弃第一个参数：

```go
sum := 0
for _, value := range array {
    sum += value
}
```

空标识有很多种用法，就像[后续章节](#空标识符)中描述的这样。

对于字符串，range 为您做了更多的事情，通过解析 UTF-8 来分解各个 Unicode 码。错误的编码消耗一个 byte 并使用 rune U+FFFD 代替（rune 是 Go 中称呼和使用单个 Unicode 码点的术语。参考[language specification](https://go.dev/ref/spec)了解更多）。对于下面的循环

```go
for pos, char := range "日本\x80語" { // \x80 is an illegal UTF-8 encoding
    fmt.Printf("character %#U starts at byte position %d\n", char, pos)
}
```

输出为

``` text
character U+65E5 '日' starts at byte position 0
character U+672C '本' starts at byte position 3
character U+FFFD '�' starts at byte position 6
character U+8A9E '語' starts at byte position 7
```

最后，Go 没有逗号运算符， ++ 和 \-\- 是语句而不是表达式。因此如果你在 for 中运行多个变量，你应该使用多重赋值（不包括 ++ 和 \-\-）。

```go
// Reverse a
for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
    a[i], a[j] = a[j], a[i]
}
```

## Switch

Go 中的 switch 比 C 中的更通用。表达式不需要是常量，甚至不需要是整数，cases 从上向下匹配，直到寻找到匹配项，如果 switch 没有表达式，则他匹配到 true。因此，习惯上可以将 if-else-if-else 链编写为 switch。

```go
func unhex(c byte) byte {
    switch {
    case '0' <= c && c <= '9':
        return c - '0'
    case 'a' <= c && c <= 'f':
        return c - 'a' + 10
    case 'A' <= c && c <= 'F':
        return c - 'A' + 10
    }
    return 0
}
```

Go 语言的 switch 不会自动执行下一个分支，但是可以使用逗号分隔的列表来组合判断条件。

```go
func shouldEscape(c byte) bool {
    switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
        return true
    }
    return false
}
```

尽管他们在 Go 中不像在其他一些类似 C 的语言中那么常见，但 break 语句可以用来中止 switch。但有时，我们需要打破外围的循环，而非 switch，在 Go 中可以通过在循环上放置标签并 "breaking" 该标签来完成。下面的例子演示了这两种用途。

```go
Loop:
    for n := 0; n < len(src); n += size {
        switch {
        case src[n] < sizeOne:
            if validateOnly {
                break
            }
            size = 1
            update(src[n])

        case src[n] < sizeTwo:
            if n+1 >= len(src) {
                err = errShortInput
                break Loop
            }
            if validateOnly {
                break
            }
            size = 2
            update(src[n] + src[n+1]<<shift)
        }
    }
```

当然， continue 语句也可以接受标签，但他仅适用于循环。

为了完成这个章节，这里有一个使用两个 switch 语句的字节切片比较函数。

```go
// Compare returns an integer comparing the two byte slices,
// lexicographically.
// The result will be 0 if a == b, -1 if a < b, and +1 if a > b
func Compare(a, b []byte) int {
    for i := 0; i < len(a) && i < len(b); i++ {
        switch {
        case a[i] > b[i]:
            return 1
        case a[i] < b[i]:
            return -1
        }
    }
    switch {
    case len(a) > len(b):
        return 1
    case len(a) < len(b):
        return -1
    }
    return 0
}
```

## Type switch

switch 也可以用来发现接口变量的动态类型。这种 type switch 使用类型断言的语法，并将关键字 type 放在括号内。如果 switch 在表达式中声明了变量，那么变量会在每个匹配项中获得相应的类型。在每个 case 项中使用相同的名称也是惯用的做法，实际上是在每个 case 中声明了名称相同但类型不同的变量。

```go
var t interface{}
t = functionOfSomeType()
switch t := t.(type) {
default:
    fmt.Printf("unexpected type %T\n", t)     // %T prints whatever type t has
case bool:
    fmt.Printf("boolean %t\n", t)             // t has type bool
case int:
    fmt.Printf("integer %d\n", t)             // t has type int
case *bool:
    fmt.Printf("pointer to boolean %t\n", *t) // t has type *bool
case *int:
    fmt.Printf("pointer to integer %d\n", *t) // t has type *int
}
```

# 函数

## 多返回值

Go 中的一个特殊特性是函数和方法可以返回多个值。这种形式可以用于改进 C 程序中的几个笨拙的惯用方法：例如通过返回 -1 来表示 EOF 或是修改一个使用指针传入的参数。

在 C 语言中，write 错误通过一个负数来表示，而错误代码隐藏在容易丢失的变量中。在 Go 中， Write 可以同时返回计数和错误：“你写入了一些字节，但没有完全写入，因为你的磁盘满了”。在 os 包中的 Write 方法的方法签名是：

```go
func (file *File) Write(b []byte) (n int, err error)
```

正如文档所说，当 n != len(b) 时，函数返回了已经写入的字节数和一个非 nil 的 error。这是一个通用的风格，查看关于错误处理的章节了解更多例子。

一个类似的可以用来替代传递指针作为参数的方法是，通过返回一个值来模拟引用参数。这里是一个简单的函数，用来从字节切片的某个定位开始抓取数字，并返回数字和数字后的定位。

```go
func nextInt(b []byte, i int) (int, int) {
    for ; i < len(b) && !isDigit(b[i]); i++ {
    }
    x := 0
    for ; i < len(b) && isDigit(b[i]); i++ {
        x = x*10 + int(b[i]) - '0'
    }
    return x, i
}
```

您可以通过这样的方式使用，扫描切片 b 中的数字：

```go
    for i := 0; i < len(b); {
        x, i = nextInt(b, i)
        fmt.Println(x)
    }
```

## 命名结果参数

在 Go 函数中返回的结果“参数”可以被命名并像其他常规变量一样使用，就像输入参数一样。当被命名时，他们会在函数开始时被初始化为其对应类型的0值；如果函数执行不带参数的返回，那么这些结果参数的当前值会被用作返回值。

命名不是强制性的，但是他们会让代码更加简短和清晰：他们是文档的一种。如果我们将 nextInt 函数的返回结果命名，那么返回的 int 的含义将变得显而易见。

```go
func nextInt(b []byte, pos int) (value, nextPos int) {
```

因为命名的返回值已经和不带参数的 return 绑定在了一起，他们可以用来简化和澄清代码。这里是使用命名结果参数的 io.ReadFull：

```go
func ReadFull(r Reader, buf []byte) (n int, err error) {
    for len(buf) > 0 && err == nil {
        var nr int
        nr, err = r.Read(buf)
        n += nr
        buf = buf[nr:]
    }
    return
}
```

## Defer

Go 中的 defer 语句可以在当前执行的函数返回前执行一次函数调用（ *deferred* 函数）。这是一个不寻常但有效的方案，可以用来处理类似于在任何函数执行路径下的资源关闭问题。典型的例子是解锁一个 mutex 或关闭一个文件。

```go
// Contents returns the file's contents as a string.
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    defer f.Close()  // f.Close will run when we're finished.

    var result []byte
    buf := make([]byte, 100)
    for {
        n, err := f.Read(buf[0:])
        result = append(result, buf[0:n]...) // append is discussed later.
        if err != nil {
            if err == io.EOF {
                break
            }
            return "", err  // f will be closed if we return here.
        }
    }
    return string(result), nil // f will be closed if we return here.
}
```

通过 defer 调用 Close 函数有两个好处。其一， 他确保了你不会忘记关闭这个文件，尤其是你之后再为函数添加更多种返回路径时。其二，他意味着 close 的位置与 open 在一起，这比将他放在函数的末尾要清晰的多。

deferred 函数的参数（或是 deferred 方法的接收者）是在 *defer* 语句执行时计算的（意味着创建一个闭包），而非是在函数被调用时计算。除了无需担心在函数执行过程中的变量值改变外，这也意味着一个 defer 语句可以创建多个延迟执行的函数。这是一个不太聪明的示例。

```go
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
}
```

Deferred 函数按照后进先出的顺序执行，因此这段代码会在函数返回时打印 4 3 2 1 0。一个更合理的例子是简单的通过程序跟踪函数调用的方法。我们可以写一些类似这样的跟踪例程：

```go
func trace(s string)   { fmt.Println("entering:", s) }
func untrace(s string) { fmt.Println("leaving:", s) }

// Use them like this:
func a() {
    trace("a")
    defer untrace("a")
    // do something....
}
```

我们可以利用 deferred 函数是在 defer 执行时计算参数这一事实来做的更好。tracing 例程可以用来设置 untracing 例程的参数。下面是例子：

```go
func trace(s string) string {
    fmt.Println("entering:", s)
    return s
}

func un(s string) {
    fmt.Println("leaving:", s)
}

func a() {
    defer un(trace("a"))
    fmt.Println("in a")
}

func b() {
    defer un(trace("b"))
    fmt.Println("in b")
    a()
}

func main() {
    b()
}
```

输出

```text
entering: b
in b
entering: a
in a
leaving: a
leaving: b
```

对于在其他编程语言中习惯了块级资源管理的程序员来说，defer 也许看起来有些特殊，但是他最有趣也是最强大的的应用恰恰来自于他是基于函数而非基于块的事实。在 panic 与 recover 章节我们会看到这种可能性的例子。

# 数据

## 通过new分配

Go 中有两个分配关键字，内置函数 new 和 make。他们做不同的事情而且用于不同的类型，这可能会令人困惑，但是规则实际上很简单。让我们先从 new 开始。这是一个内建的用于分配内存的函数，但与一些其他编程语言中的同名函数不同，他并不会初始化内存，而只是将内存值置零。也就是说，new(T) 分配了一块零值的内存用来放置类型 T 并返回他的指针，一个类型为 *T 的值。在 Go 术语中，他返回了一个指向新分配的零值 T 的指针。

由于 new 返回的内存值总是 0，因此在设计您的数据结构时，将 0 值作为一个合理的初始化值就非常有帮助。这意味着数据结构的使用者可以通过 new 来创建他并立刻使用。例如 bytes.Buffer 的文档表示“零值的 Buffer 是一个可以被使用的空 buffer”。类似的，sync.Mutex 也没有显式的构造函数或是初始化方法。取而代之的是，零值的 sync.Mutex 被定义为一个未锁定的 mutex。

零值有效的设计方式是可以传递的，考虑下述类型声明。

```go
type SyncedBuffer struct {
    lock    sync.Mutex
    buffer  bytes.Buffer
}
```

类型为 SyncedBuffer 的值也可以在被分配或声明后立即使用。在下面的片段里，p 和 v 都可以在无需更多参数的情况下正确使用。

```go
p := new(SyncedBuffer)  // type *SyncedBuffer
var v SyncedBuffer      // type  SyncedBuffer
```

## 构造器与复合字面量(composite literals)

有些情况下无法通过零值初始化，这时就需要构造器，例如这个 os 包中的例子。

```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := new(File)
    f.fd = fd
    f.name = name
    f.dirinfo = nil
    f.nepipe = 0
    return f
}
```

这里有很多的赋值操作。我们可以简单的使用复合字面量，这是一种在每次执行时创建一个新实例的表达式。

```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := File{fd, name, nil, 0}
    return &f
}
```

请注意，与 C 不同，返回局部变量的地址是完全可行的；在函数返回后与变量关联的存储依然存在。实际上，每次获取复合字面量地址时都会分配一个新的实例，因此我们可以合并最后两行。

```go
    return &File{fd, name, nil, 0}
```

复合字面量中的字段必须按序且全部存在。然而，通过使用 field:value 对标记元素，初始值设定可以以任何顺序出现，缺失的字段保留位各自的零值。因此我们可以

```go
    return &File{fd: fd, name: name}
```

作为一个特殊情况，如果复合字面量没有包括任何字段，他会创建当前类型的零值对象。表达式 new(File) 和 &File{} 是等效的。

复合字面量也可以用来创建数组、切片和 maps，其中的字段标签被视为索引或 map 中的键。在这些示例中，只要 Enone、Eio、Einval 的值不同，无论他们的值如何，初始化都会生效。

```go
a := [...]string   {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
s := []string      {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
m := map[int]string{Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
```

## 通过make分配

回到内存分配的话题。内置函数 make(T, args) 提供了一个与 new(T) 不同的用途。他只用来创建切片、maps 和 channels，而且他返回一个已经被初始化的（非零值）的 T 的值（而非 *T）。区别的原因是这三种类型在底层都是对必须被初始化的某种数据结构的引用。例如切片，由三个元素组成，指向数据的指针（数据是内置的数组）、长度和容量，在这些元素被初始化前，切片的值是 nil。对于切片、maps 和 channels，make 初始化了他们内部的数据结构而且提供了可用的值。对于，

```go
make([]int, 10, 100)
```

分配了一个长度为 100 int 的数组，随后创建切片结构，长度为 10，容量为 100，指向数组最初的 10 个元素。（在创建切片时，可以不手动指定容量；参考切片章节了解更多信息）。相反，new([]int) 返回一个新分配的，值为零的切片结构的指针，也就是一个指向 nil 切片值的指针。

这些例子阐明了 new 和 make 的区别。

```go
var p *[]int = new([]int)       // allocates slice structure; *p == nil; rarely useful
var v  []int = make([]int, 100) // the slice v now refers to a new array of 100 ints

// Unnecessarily complex:
var p *[]int = new([]int)
*p = make([]int, 100, 100)

// Idiomatic:
v := make([]int, 100)
```

请记住，make 用于 maps，切片和 channels，并不返回指针。如果想要显式的获得指针，需要使用 new 或是显式的获取变量的地址。

## 数组

数组在进行详细的内存规划时很有用，而且有时候可以用来避免分配动作，但是大部分时候他们还是用于作为切片的一部分使用，也就是下一个章节的主题。为了给下一个主题打下基础，这里简单介绍一下数组。

这里有一些 Go 中数组和 C 中数组的主要区别。在 Go 中，

- 数组是值，将一个数组赋值给另一个会复制所有的元素。
- 特别是，将数组传递给函数，他会收到数组的副本，而不是数组的指针。
- 数组的尺寸使其类型的一部分。[10]int 和 [20]int 是不同的类型。

使用值传递可能会有用，但是开销也很大；如果你想要和 C 相似的表现方式和效率，你可以传递一个数组的指针。

```go
func Sum(a *[3]float64) (sum float64) {
    for _, v := range *a {
        sum += v
    }
    return
}

array := [...]float64{7.0, 8.5, 9.1}
x := Sum(&array)  // Note the explicit address-of operator
```

但是这也不是 Go 中的惯用方式，Go 程序员使用切片。

## 切片

切片封装了数组，提供了一个更加通用、强大、方便的处理序列数据的接口。除了例如变换矩阵这种有显式的维度的情况外，在 Go 中大部分关于序列数据的编程都是通过切片而非数组完成的。

切片持有了对其下层数组的引用，如果你将一个切片赋值给另一个，他们将会引用同一个数组。如果函数使用切片作为参数，则调用者可以看到函数对切片中元素的修改，类似于传递了下层数组的指针。因此，Read 函数可以接收切片作为参数，而非指针和计数；切片中的长度设置了要读取数据内容的上限。这里是 os 包中 File 类型的 Read 方法的方法签名：

```go
func (f *File) Read(buf []byte) (n int, err error)
```

该方法返回读取的字节数和可能的错误值。如果想要将前 32 个字节的内容读取到一个大容量的 buffer（名为 buf）中，可以对这个 buffer 进行切片（此处为动词）。

```go
    n, err := f.Read(buf[0:32])
```

这种切片是常见且高效的。事实上，暂且不考虑执行效率，下面的代码片段也可以将前 32 个字节读取到 buffer 中。

```go
    var n int
    var err error
    for i := 0; i < 32; i++ {
        nbytes, e := f.Read(buf[i:i+1])  // Read one byte.
        n += nbytes
        if nbytes == 0 || e != nil {
            err = e
            break
        }
    }
```

切片的长度可以在其底层数据限制的范围内任意修改；只要切取他自身的一部分并赋值即可。切片的容量可以通过内建函数 cap 访问，报告了切片可用的最大长度。这里是一个向切片添加数据的函数。如果数据超出了容量限制，切片会采用重新分配的内存。最终的结果是函数返回的切片。设计这个函数时巧妙利用了如下事实：对 nil 切片使用 len 和 cap 操作是合法的，且会返回 0。

```go
func Append(slice, data []byte) []byte {
    l := len(slice)
    if l + len(data) > cap(slice) {  // reallocate
        // Allocate double what's needed, for future growth.
        newSlice := make([]byte, (l+len(data))*2)
        // The copy function is predeclared and works for any slice type.
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:l+len(data)]
    copy(slice[l:], data)
    return slice
}
```

最后我们必须返回切片，因为虽然 Append 可以改变切片中的元素，但切片本身（运行时的数据结构，包括指针，长度和容量）是按值传递的。

向切片添加元素是一个常用操作，因此内建函数 append 实现了这一功能。为了理解这个函数的设计，我们需要掌握更多的信息，因此我们会在稍后讨论他。

## 二维切片

Go 中的数组和切片是一维的。为了等效的实现二维数组或二维切片，必须定义 数组的数组 或是 切片的切片，类似于：

```go
type Transform [3][3]float64  // A 3x3 array, really an array of arrays.
type LinesOfText [][]byte     // A slice of byte slices.
```

因为切片的长度是可变的，因此每个内部切片的长度不同是可能的。这是一个很常见的情况，如果我们的 文本行 的例子：每一行都有独立的长度。

```go
text := LinesOfText{
    []byte("Now is the time"),
    []byte("for all good gophers"),
    []byte("to bring some fun to the party."),
}
```

有时需要分配二维切片，例如在处理逐行像素扫描时。有两种方式可以实现这一目标。一种是为每个切片独立进行分配；另一种是分配一个独立的数组，然后将各个切片指向其中不同的区域。使用哪一种方式取决于您的应用程序。如果您的切片可能会增大或缩小，那应该独立分配切片，避免覆盖到下一行的内容；如果不是，单次分配一个数据来使用可能会更高效。作为参考，这里提供了两种方式的简单实现。第一种，每次一行的方式：

```go
// Allocate the top-level slice.
picture := make([][]uint8, YSize) // One row per unit of y.
// Loop over the rows, allocating the slice for each row.
for i := range picture {
    picture[i] = make([]uint8, XSize)
}
```

现在是单次分配，切为多行：

```go
// Allocate the top-level slice, the same as before.
picture := make([][]uint8, YSize) // One row per unit of y.
// Allocate one large slice to hold all the pixels.
pixels := make([]uint8, XSize*YSize) // Has type []uint8 even though picture is [][]uint8.
// Loop over the rows, slicing each row from the front of the remaining pixels slice.
for i := range picture {
    picture[i], pixels = pixels[:XSize], pixels[XSize:]
}
```

## Maps

Maps 是一种方便且强大的内建数据结构，他将一种类型的值（键）关联到另一种类型的值（值）。键可以是支持相等操作的任意数据类型，例如整数、浮点数、复数、字符串、指针、接口（只要其中的动态类型支持相等）、结构体或数组。切片不能用作 map 的键，因为没有为其定义相等操作。与切片类似，maps 持有了对下层数据结构的引用。如果你将 map 传入一个函数并且在函数中改变了 map 的内容，调用者也会看到这个改变。

Maps 可以使用常规的复合字面量语法，加上冒号分隔键值对来构建，因此在初始化期间构建他们很容易。

```go
var timeZone = map[string]int{
    "UTC":  0*60*60,
    "EST": -5*60*60,
    "CST": -6*60*60,
    "MST": -7*60*60,
    "PST": -8*60*60,
}
```

对 map 分配值或获取值的语法看起来和对数组或切片的操作相同，只是索引不必须是整数。

```go
offset := timeZone["EST"]
```

尝试获取 map 中不存在键映射的值会返回该类型的零值。例如，如果 map 包含整数，查询一个不存在的键会返回 0。可以使用 bool 作为值的 map 来实现 set。将每个键的值都设置为 true，之后可以通过索引来简单的判断该键是否存在。

```go
attended := map[string]bool{
    "Ann": true,
    "Joe": true,
    ...
}

if attended[person] { // will be false if person is not in the map
    fmt.Println(person, "was at the meeting")
}
```

有时您需要去区分缺失条目或是零值。究竟是是否有 UTC 的值，或者说 0 只是因为他并没有在 map 中设置？你可以使用多重赋值的方式分辨。

```go
var seconds int
var ok bool
seconds, ok = timeZone[tz]
```

出于一些显然的原因，这被称为 comma ok 用法。在下面的例子中，如果 tz 存在，seconds 将会被赋值而且 ok 的值为 true；如果 tz 不存在，seconds 的值将会是 0 而且 ok 的值会是 false。下面是一个将这些内容和一个错误日志组合到一起的函数：

```go
func offset(tz string) int {
    if seconds, ok := timeZone[tz]; ok {
        return seconds
    }
    log.Println("unknown time zone:", tz)
    return 0
}
```

如果不关心值的内容，只想测试键是否存在，您可以使用空标识符 (_) 代替值的变量。

```go
_, present := timeZone[tz]
```

要删除 map 中的条目，可以使用内建的 delete 函数，他需求的参数是 map 和要被删除的键。即使要删除键已经不在 map 中，这个操作也是安全的。

```go
delete(timeZone, "PDT")  // Now on Standard Time
```

## 打印

Go 中的格式化输出采用了和 C 中的 printf 系列相似但更加丰富和通用的实现。这些函数位于 fmt 包中且拥有大写的名称：fmt.Printf，fmt.Fprintf，fmt.Sprintf 等等。字符串系列函数（Sprintf等）返回一个字符串，而非填充某个指定的 buffer。

您不需要提供格式化字符串。对于每个 Printf，Fprintf 和 Sprintf 都存在另外两个与之对应的函数，例如 Print 和 Println。这些函数不接收格式化字符串，但为他们的每个参数提供默认格式。Println 会在两个参数间加入空格，而且在输出内容末尾添加换行符，Print 则是只有当两个连续的参数不是字符串时才添加空格。在这个例子中每一行都会提供同样的输出。

```go
fmt.Printf("Hello %d\n", 23)
fmt.Fprint(os.Stdout, "Hello ", 23, "\n")
fmt.Println("Hello", 23)
fmt.Println(fmt.Sprint("Hello ", 23))
```

fmt.Fprint 系列的格式化输出函数接收任何实现了 io.Writer 接口的对象作为其第一个参数；os.Stdout 和 os.Stderr 是最常见的用法。

从现在开始事情变得和 C 不同。首先，%d 这样的数字格式不采用符号或大小标志；相反，打印例程使用参数的类型来决定这些属性。

```go
var x uint64 = 1<<64 - 1
fmt.Printf("%d %x; %d %x\n", x, x, int64(x), int64(x))
```

输出

```text
18446744073709551615 ffffffffffffffff; -1 -1
```

如果您只想要默认转换，例如输出十进制整数，您可以使用万能格式 %v （含义是 "value"）；输出将会是 Print 和 Println 默认产生的结果。此外，这个格式可以用来打印任何值，甚至数组、切片、结构体和 maps。这里是打印上一章节定义的时区 map 的语句。

```go
fmt.Printf("%v\n", timeZone)  // or just fmt.Println(timeZone)
```

输出是这样的：

```text
map[CST:-21600 EST:-18000 MST:-25200 PST:-28800 UTC:0]
```

对于 maps，Printf 系列函数按字典顺序对输出进行排序。

在打印结构体时，使用 %+v 可以将字段值和名称共同输出，使用 %#v 则可以输出 Go 格式中全量的信息。

```go
type T struct {
    a int
    b float64
    c string
}
t := &T{ 7, -2.35, "abc\tdef" }
fmt.Printf("%v\n", t)
fmt.Printf("%+v\n", t)
fmt.Printf("%#v\n", t)
fmt.Printf("%#v\n", timeZone)
```

输出

```text
&{7 -2.35 abc   def}
&{a:7 b:-2.35 c:abc     def}
&main.T{a:7, b:-2.35, c:"abc\tdef"}
map[string]int{"CST":-21600, "EST":-18000, "MST":-25200, "PST":-28800, "UTC":0}
```

（注意 & 符号。）当输出 string 或 []byte 类型的值时，可以使用 %q 产生带引号的输出。如果可以的话，%#q 会使用反引号输出。（%q 也可以用于整数和 runes，输出一个单引号的 rune。）同样，%x 对 string、byte 数组、byte 切片和对整数有同样的效果，产生一个长十六进制字符串，如果在该格式中添加空格（% x），输出时会在字节间加入空格。

另一种方便的格式时 %T，他会输出值的类型。

```go
fmt.Printf("%T\n", timeZone)
```

输出

```text
map[string]int
```

如果您想要控制自定义类型的默认输出格式，只需为该类型定义方法签名为 String() 的方法。对于一个简单的例子 T，看起来可能是这样。

```go
func (t *T) String() string {
    return fmt.Sprintf("%d/%g/%q", t.a, t.b, t.c)
}
fmt.Printf("%v\n", t)
```

打印出的内容

```text
7/-2.35/"abc\tdef"
```

(如果您需要打印 T 的类型和指向 T 的指针，String 函数的接收者必须是值类型；这个例子中使用指针是因为这样执行效率更高，也更复合结构体类型的习惯。查看后续章节[指针或是值](#指针或是值)了解更多信息。)

在我们的 String 方法中可以调用 Sprintf，因为打印例程是完全可重入的，而且可以通过这种方式包裹。然而，关于这种方法有一个非常重要的细节需要了解：不要使用会调用您的 String 方法的 Spintf 来构建您的 String 方法，这会导致您的 String 方法被无限调用。这种情况可能在您的 Sprintf 尝试直接将方法接收者直接作为 string 输出时发生，从而再次调用该方法。这是一个常见的但容易解决的问题，比如这个例子。

```go
type MyString string

func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", m) // Error: will recur forever.
}
```

修复起来也很简单：将参数转换为基本的 string 类型，转换后的参数不会调用这个方法。

```go
type MyString string
func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", string(m)) // OK: note conversion.
}
```

在[初始化](#初始化)部分，我们将看到另一种避免这种递归的技术。

另一种打印技术是将打印例程的参数直接传递给另一个例程。Printf 的方法签名使用 ...interface() 作为最后一个参数，用来指定任意数量（以及任意类型）的参数都可以出现在 format 之后。

```go
func Printf(format string, v ...interface{}) (n int, err error) {
```

在函数 Printf 中，v 的作用类似于类型为 []interface{} 的变量，但是如果将其传递到另一个可变参数的函数中，他则可以当作常规的参数列表使用。这里是我们之前用过的 log.Println 函数的实现。他直接将参数传递到 fmt.Sprintln 中，从而进行实际的格式化工作。

```go
// Println prints to the standard logger in the manner of fmt.Println.
func Println(v ...interface{}) {
    std.Output(2, fmt.Sprintln(v...))  // Output takes parameters (int, string)
}
```

我们在嵌套调用 Sprintln 时在参数 v 之后写上 ...，用来告诉编辑器将 v 作为一个参数列表对待；否则他会将 v 作为一个切片类型的参数传递。

关于打印的内容比我们在这里介绍的还要多。有关详细信息，请参阅 fmt 包的 godoc 文档。

顺便提一句，...参数可以指定类型，例如使用 ...int 实现的最小值函数，用来选取整数列表中的最小值：

```go
func Min(a ...int) int {
    min := int(^uint(0) >> 1)  // largest int
    for _, i := range a {
        if i < min {
            min = i
        }
    }
    return min
}
```

## Append

现在我们需要解释之前缺失的碎片，关于内建函数 append 的设计。append 的方法签名和之前我们自定义的 Append 函数不同。示意一下的话，他看起来类似这样：

```go
func append(slice []T, elements ...T) []T
```

此处的 T 表示任何给定的类型。你不能在 Go 中编写一个由调用者决定 T 类型的函数。这也就是为何 append 作为内建函数的原因：他需要编译器的支持。

append 做的事情是将元素加入到切片的末尾并返回结果。需要返回结果的原因和我们之前手写的 Append 一样，底层的数组可能已经被改变。这里有个而简单的例子

```go
x := []int{1,2,3}
x = append(x, 4, 5, 6)
fmt.Println(x)
```

输出 [1 2 3 4 5 6]。因此 append 的工作方式有点像 Printf，因为他可以接收任意数量和任意类型的参数。

但如果我们只想做和我们自定义 Append 一样的事情，把一个切片添加到另一个切片之后呢？简单：在调用时使用 ...，就像我们在之前的 Output 调用时做的那样。这个片段会产生和上一个完全相同的输出。

```go
x := []int{1,2,3}
y := []int{4,5,6}
x = append(x, y...)
fmt.Println(x)
```

如果缺少了 ...，这段代码会因为类型错误而无法编译，因为 y 的类型并不是 int。

# 初始化

尽管从表面上看它与 C 或 C++ 中的初始化没有太大区别，但 Go 中的初始化更强大。可以在初始化期间构建复杂的结构，并且可以正确处理初始化对象之间（甚至不同包之间）的排序问题。

## 常量

在 Go 中，常量的意思是——常量。他们是在编译时创建的，即使是在函数中被定义为局部变量也是如此，并且只能是数字、字符（rune）、字符串或是布尔值。由于编译时的限制，定义常量的表达式必须是常量表达式，可由编译器计算。例如，1<<3 是一个常量表达式，而 math.Sin(math.Pi/4) 则不是，因为对函数 math.Sin 的调用需要在运行时发生。

在 Go 中，枚举常量是使用 iota 枚举器创建的。由于 iota 可以是表达式的一部分，并且表达式可以隐式重复，因此很容易构建复杂的值集。

```go
type ByteSize float64

const (
    _           = iota // ignore first value by assigning to blank identifier
    KB ByteSize = 1 << (10 * iota)
    MB
    GB
    TB
    PB
    EB
    ZB
    YB
)
```

将类似 String 这样的方法附加到任何用户自定义类型的能力使得任何值都可以在打印时格式化输出自己。虽然最常见的用法是对结构体的应用，这种技术对于标量类型（例如浮点数表示的字节大小）也很有用。

```go
func (b ByteSize) String() string {
    switch {
    case b >= YB:
        return fmt.Sprintf("%.2fYB", b/YB)
    case b >= ZB:
        return fmt.Sprintf("%.2fZB", b/ZB)
    case b >= EB:
        return fmt.Sprintf("%.2fEB", b/EB)
    case b >= PB:
        return fmt.Sprintf("%.2fPB", b/PB)
    case b >= TB:
        return fmt.Sprintf("%.2fTB", b/TB)
    case b >= GB:
        return fmt.Sprintf("%.2fGB", b/GB)
    case b >= MB:
        return fmt.Sprintf("%.2fMB", b/MB)
    case b >= KB:
        return fmt.Sprintf("%.2fKB", b/KB)
    }
    return fmt.Sprintf("%.2fB", b)
}
```

表达式 YB 会输出为 1.00YB，而 ByteSize(1e13) 则会输出为 9.09TB。

这里使用 Sprintf 来实现 ByteSize 类型的 String 方法是安全的（不会产生无限调用），并不是因为数据转换，而是因为他使用 %f 作为 Sprintf 的参数，这不是字符串格式的：Sprintf 只会在需要字符串值时调用 String 方法，而 %f 表示需要的是浮点数的值。

## 变量

变量可以像常量一样被初始化，但是其初始化表达式可以在运行时计算。

```go
var (
    home   = os.Getenv("HOME")
    user   = os.Getenv("USER")
    gopath = os.Getenv("GOPATH")
)
```

## init函数

最后，每个源文件都可以定义自己的 无参数（niladic） init 函数，用来设置任何他们所需的状态。（实际上每个文件可以有多个 init 函数。）最后的最后：init 会在包中声明的所有变量完成初始化计算后执行，而这些变量的初始化计算则会在所有引入的包已经完成初始化后执行。

除了执行不能用声明表示的初始化动作外，init 函数的一个常见用途是在程序开始前确认或修复程序状态的正确性。

```go
func init() {
    if user == "" {
        log.Fatal("$USER not set")
    }
    if home == "" {
        home = "/home/" + user
    }
    if gopath == "" {
        gopath = home + "/go"
    }
    // gopath may be overridden by --gopath flag on command line.
    flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
}
```

# 方法

## 指针或是值

如同我们之前在 Bytesize 中看到的，方法可以为任何命名类型定义（除了指针或接口）；方法的接收者不必是一个结构体。

在之前我们对切片的讨论中，我们写了 Append 函数。我们可以将他替换为切片的方法。为了实现这个目标，首先我们声明一个命名的类型，这样我们可以将方法绑定在上面，之后我们将该类型的值作为方法的接收者。

```go
type ByteSlice []byte

func (slice ByteSlice) Append(data []byte) []byte {
    // Body exactly the same as the Append function defined above.
}
```

这个方法仍然需要配返回更新后的切片。我们可以通过重新定义方法，采用 ByteSlice 的指针作为接收者，来消除这种不便，调用这个方法会覆盖调用者的切片。

```go
func (p *ByteSlice) Append(data []byte) {
    slice := *p
    // Body as above, without the return.
    *p = slice
}
```

实际上，我们甚至可以做的更好。如果我们修改我们的函数，让他看起来像标准的 Write 方法，比如

```go
func (p *ByteSlice) Write(data []byte) (n int, err error) {
    slice := *p
    // Again as above.
    *p = slice
    return len(data), nil
}
```

之后，*ByteSlice 类型满足了基础的 io.Writer 接口，这样就很方便。例如，我们可以使用 print 写入。

```go
    var b ByteSlice
    fmt.Fprintf(&b, "This hour has %d days\n", 7)
```

我们传递 ByteSlice 的地址是因为只有 *ByteSlice 类型满足了 io.Wirter 的接口条件。使用指针或是值作为接收者的规则是这样的，值的方法可以被指针或是值调用，而指针的方法只能被指针调用。

这条规则的出现是因为指针的方法可以修改其接收者，在值上调用这种方法会导致方法接收到值的复制，因此任何修改都会失效。因此，该语言不允许出现这种错误。不过，有一个方便的例外。当值本身可寻址时，Go 通过在值前自动插入 & 照顾了使用值调用指针方法的情况。在我们的例子中，变量 b 是可寻址的，因此我们可以使用 b.Write 调用他的 Write 方法。编译器会帮助我们自动重写为 (&b).Write。

顺带一提，通过 Write 像字节切片写入数据的想法是 bytes.Buffer 包实现的核心。

# 接口和其他类型

## 接口

Go 中的接口提供了一个指定对象行为的方法：如果他可以实现这个功能，那么他就可以在这里使用。我们之前已经看到了几个简单的例子；自定义打印可以通过 String 方法实现，而 Fprintf 可以向任何实现了 Write 方法的对象输出内容。Go 中的接口往往只定义了一到两个方法，而且通常会根据方法指定一个名称，例如 io.Writer 表示任何 Write 接口实现。

一个类型可以实现多个接口。例如，如果一个集合实现了 sort.Interface（其中包含了 Len，Less(i, j int) bool，Swap(i, j int)），那么他就可以被 sort 包中的例程排序，同时他也可以有一个自定义的格式化器。在下面这个定制的例子中，Sequence 同时满足了这两个条件。

```go
type Sequence []int

// Methods required by sort.Interface.
func (s Sequence) Len() int {
    return len(s)
}
func (s Sequence) Less(i, j int) bool {
    return s[i] < s[j]
}
func (s Sequence) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

// Copy returns a copy of the Sequence.
func (s Sequence) Copy() Sequence {
    copy := make(Sequence, 0, len(s))
    return append(copy, s...)
}

// Method for printing - sorts the elements before printing.
func (s Sequence) String() string {
    s = s.Copy() // Make a copy; don't overwrite argument.
    sort.Sort(s)
    str := "["
    for i, elem := range s { // Loop is O(N²); will fix that in next example.
        if i > 0 {
            str += " "
        }
        str += fmt.Sprint(elem)
    }
    return str + "]"
}
```

## 转换

Sequence 类型的 String 方法重复了 Sprint 输出切片的工作。（而且复杂度为 O(N²)，这很糟糕）。如果我们在调用 Sprint 之前将他转换为普通的 []int，那么我们的工作量会减少，运行速度也会提升。

```go
func (s Sequence) String() string {
    s = s.Copy()
    sort.Sort(s)
    return fmt.Sprint([]int(s))
}
```

这个方法是另一个通过转换技术在 String 方法中安全调用 Sprintf 的例子。因为这两种类型（Sequence 和 []int）在忽略名称的情况下实际上是相同的，因此这种转换是合法的。这种转换不会创建新的值，他只是暂时的将值作为一个新的类型来使用。（还有一种合法的转换，例如将整数转换为浮点数，过程中会创建新的值。）

通过转换类型来使用不同的方法是一种 Go 程序中的常见用法。作为示例，我们可以使用 sort.IntSlice 将上文中的例子改变为：

```go
type Sequence []int

// Method for printing - sorts the elements before printing
func (s Sequence) String() string {
    s = s.Copy()
    sort.IntSlice(s).Sort()
    return fmt.Sprint([]int(s))
}
```

现在，不再是让 Sequence 实现多个接口（sorting 和 printing），我们通过使用将数据转换为多种类型的能力（Sequence，sort.IntSlice 和 []int），每种类型可以解决一部分工作。这种方式在实践中并不常用，但是可能会很有效。

## 接口转换与类型断言

[Type switch](#Type-switch) 是一种类型转换的形式：获取一个接口，对于 switch 中的每个 case，将接口“转换”为 case 对应的类型。这里是一个 fmt.Printf 如何通过类型转换将值转换为字符串的示例。如果值早已是 string，我们获取该接口持有的 string 本身的值，当值有 String 方法是，我们获取该方法的输出。

```go
type Stringer interface {
    String() string
}

var value interface{} // Value provided by caller.
switch str := value.(type) {
case string:
    return str
case Stringer:
    return str.String()
}
```

第一个 case 匹配一个具体的类型；第二个 case 将接口转换成另一个接口。这样混合不同种类的使用是完全可行的。

当只有一种我们关心的类型时呢？如果我们提前知道值是 string 类型，而且我们只想将他提取出来呢？使用只有一个 case 的 type switch 是可以的，但 *类型断言* 也可以。类型断言使用一个接口值，从中提取出一个类型明确的值。该语法借鉴了 type switch，但是使用明确的类型而非 type 关键字：

```go
value.(typeName)
```

获得的结果就是一个符合指定的 typeName 类型的值。新的类型要么是接口持有的具体类型，要么是值可以转换的另一个接口类型。为了提炼我们已经知道的类型为 string 的值，我们可以：

```go
str := value.(string)
```

但是如果实际上值并非 string，程序会产生一个运行时错误并崩溃。为了防止这种情况，可以使用 "comma, ok" 方式来测试，判断值到底是否属于 string 类型：

```go
str, ok := value.(string)
if ok {
    fmt.Printf("string value is: %q\n", str)
} else {
    fmt.Printf("value is not a string\n")
}
```

如果类型断言失败，str 依然会是 string 类型的变量，但是他的值为零值，也就是一个空字符串。

作为补充说明，这里是一个使用类型断言和 if-else 实现的语句，达到了和本章开始时 type switch 相同的效果。

```go
if str, ok := value.(string); ok {
    return str
} else if str, ok := value.(Stringer); ok {
    return str.String()
}
```

## 概论

如果一个类型的存在仅仅是为了实现某个接口，而且不会导出除了接口方法外的任何方法，那么这个类型本身也无需被导出。仅导出接口清晰的表明了该值没有超出接口范围的行为能力。这也避免了在常见方法的各个实例上重复编写文档的必要。

在这种情况下，构造器应该返回接口类型的值而非实际实现类型的值。例如：在 hash 库中，crc32.NewIEEE 和 adler32.New 都返回了 hash.Hash32 接口的值。在 Go 中用 CRC-32 算法替换 Adler-32 只需要改变构造器中的调用；代码的其他部分都不会受到算法改变的影响。

类似的方法允许将各种加密包中的流式密码算法与他们相关的块状加密分开。crypto/cipher 包中的 Block 接口指定了块状加密的行为，它提供单个数据块的加密。然后，与 bufio 包类比，实现该接口的 cipher 包可用于构造由 Stream 接口表示的流式加密，而无需知道块状加密的细节。

crypto/cipher 中的接口看起来是这样：

```go
type Block interface {
    BlockSize() int
    Encrypt(dst, src []byte)
    Decrypt(dst, src []byte)
}

type Stream interface {
    XORKeyStream(dst, src []byte)
}
```

这是 counter mode (CTR) 流的定义，他将块状加密转换为流式加密；请注意，关于块状加密的细节已经全部被抽象处理：

```go
// NewCTR returns a Stream that encrypts/decrypts using the given Block in
// counter mode. The length of iv must be the same as the Block's block size.
func NewCTR(block Block, iv []byte) Stream
```

NewCTR 不止是适用于某一种指定的加密算法或是数据源，而是可以适配任何 Block 接口的实现和任何数据流。因为他们返回的是接口类型的值，更换 CRT 加密模式是完全本地化的更改。构造器的调用必须修改，但是由于其他的代码仅仅将结果作为 Stream 看待，他们则不会受到影响。

## 接口与方法

由于几乎所有类型都可以绑定方法，因此所有类型也都可以用来成为接口实现。http 包就是一个很好的例子，其中定义了 Handler 接口。所有实现了 Handler 接口的对象都可以用来处理 HTTP 请求。

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

ResponseWriter 自身就是一个接口，提供了需要向客户端返回响应的方法的入口。这些方法包括了基础的 Write 方法，因此 http.ResponseWriter 可以用在任何 io.Writer 可用的地方。Request 是一个包含了已经解析好的客户端请求的结构体。

简单起见，让我们忽略 POSTs，假装 HTTP 请求总是 GETs；这种简化不会影响到 handlers 的代码逻辑。这里有一个 handler 实现的小例子，可以用来统计页面被访问的次数。

```go
// Simple counter server.
type Counter struct {
    n int
}

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ctr.n++
    fmt.Fprintf(w, "counter = %d\n", ctr.n)
}
```

（继续我们的主题，请注意 Fprintf 是如何向 http.ResponseWriter 输出内容的。）在实际的服务器中，对 ctr.n 的访问需要进行并发保护。参考 sync 和 atomic 包来获取建议。

作为引用，这里是如何将这样一个服务添加到 URL 树上。

```go
import "net/http"
...
ctr := new(Counter)
http.Handle("/counter", ctr)
```

但是为什么我们需要将 Counter 定义为结构体呢？使用整数就足够满足所有的需求了。（方法的接收者需要设置为指针，这样数字的增长才对调用者可见。）

```go
// Simpler counter server.
type Counter int

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    *ctr++
    fmt.Fprintf(w, "counter = %d\n", *ctr)
}
```

如果您的程序有一些内部状态想要得知页面已经被访问的话？可以将 channel 绑定到服务中。

```go
// A channel that sends a notification on each visit.
// (Probably want the channel to be buffered.)
type Chan chan *http.Request

func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ch <- req
    fmt.Fprint(w, "notification sent")
}
```

最后，假设我们想要在 /arg 上显示服务器启动时使用的参数。编写一个打印参数的函数很简单。


```go
func ArgServer() {
    fmt.Println(os.Args)
}
```

如何将其变为 HTTP 服务呢？我们可以将将 ArgServer 设置成某些我们不关注的类型的方法，但是有一个更清晰的实现方式。由于我们可以为除了指针和接口外的所有类型定义方法，我们也可以为函数定义方法。http 包中包含了这样的定义：

```go
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers.  If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler object that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, req).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
    f(w, req)
}
```

HandlerFunc 是一个有 ServerHTTP 方法的类型，因此该类型的数据实现了 Handler 接口，可以处理 HTTP 请求。观察这个方法的实现：接收者是一个函数 f，而方法调用了 f。这可能看起来有点奇怪，但是本质上和用 channel 作为接收者然后在方法中向 channel 发送数据没什么区别。

为了将 ArgServer 变成 HTTP 服务，首先我们改变他的方法签名。

```go
// Argument server.
func ArgServer(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(w, os.Args)
}
```

ArgServer 现在有了和 HanderFunc 相同的方法签名，所以他可以被类型转换为 HanderFunc 从而使用 HanderFunc 的方法，就像我们将 Sequence 转换为 IntSlice 从而使用 IntSlice.Sort 一样。设置的代码很简单：

```go
http.Handle("/args", http.HandlerFunc(ArgServer))
```

当有人访问页面 /args 时，此处的处理器是 HandlerFunc 类型的 ArgServer。HTTP 服务器会调用处理器的 ServeHTTP 方法，而 ArgServer 作为接收者，实际上会调用 ArgServer（通过 HandlerFunc.ServeHTTP 中的 f(w,rea)）。相关的参数结果也会随之返回。

在这一章节中我们使用多种方式实现 HTTP 服务，包括结构体、整数、channel 和 函数，这一切都是因为接口只是方法的集合，几乎所有类型都可以成为接口的实现。

# 空标识符

在之前的 [for range 循环](#For) 和 [maps](#Maps) 的内容中，我们已经几次提到了关于空标识符的内容。空标识符可以被声明为任何类型的任何数据，之后该数据则会被无害的丢弃。这有点像在 Unix 中向 /dev/null 文件写入内容：他提供了一个需要变量占位符但是实际值又无关紧要的场景下的只写的值。他的用途比我们之前见到的还要更加广泛。

## 多重赋值中的空标识符

在 for range 循环中使用空标识符其实是某种通用解决方案的特例：在多重赋值中使用空标识符。

如果在赋值语句的左侧需要多个变量，但是其中的某个变量实际上又不会被系统使用，那么使用空标识符就可以避免我们去创建一个无效的变量，也可以清晰的表明此处变量的值会被丢弃。例如，当调用的函数可以同时提供返回值和错误值，但我们仅需要错误值时，可以使用空标识符丢弃无关紧要的返回值。

```go
if _, err := os.Stat(path); os.IsNotExist(err) {
    fmt.Printf("%s does not exist\n", path)
}
```

偶尔你会看到有代码通过丢弃错误的方式来忽略对他们的处理；这是一种很糟糕的实践。请确保总是检查错误的值，提供错误返回是有原因的。

```go
// Bad! This code will crash if path does not exist.
fi, _ := os.Stat(path)
if fi.IsDir() {
    fmt.Printf("%s is a directory\n", path)
}
```

## 未使用的引入和变量

引入一个包或声明一个变量后不去使用是一种错误。未使用的引入会导致程序膨胀，编译速度变慢，当一个变量被初始化但没有使用时，至少他会造成计算性能的浪费，而且可能会表示某处存在更大的 bug。当一个程序处在活跃的开发阶段时，未使用的引入和变量会经常出现，如果只是为了继续编译而删除他们，而之后又需要重新引入或声明，这很让人懊恼。空标识符提供了对这个问题的解决方案。

这个半成品程序包括两个未使用的引入（fmt 和 io）和一个未使用的变量（fd），因此他无法被编译，但是假设我们想要查看至今为止的代码是否正确。

```go
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
}
```

为了消除未使用引入的报错，可以使用空标识符从引入的包中引用一个变量。类似的，将 fd 赋值给空标识符会消除未使用变量的错误。这个版本的程序就可以编译了。

```go
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

var _ = fmt.Printf // For debugging; delete when done.
var _ io.Reader    // For debugging; delete when done.

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
    _ = fd
}
```

方便起见，为了消除引入错误而进行的全局声明应该集中在 imports 代码块后并且被注解，这样既可以方便的找到他们，又可以提醒我们之后将他们清理掉。

## 为了潜在作用而引入

前文中提到的未使用的引入例如 fmt 或 io 最终会被使用或被移除：空赋值表示了代码仍在开发过程中。但有时，不去显式的使用一个包，而仅仅是为了他的潜在作用而引入他也是有效的。例如，[net/http/pprof](https://go.dev/pkg/net/http/pprof/) 包在他的 init 函数中注册了提供 debug 信息的 HTTP 接口。他有一个导出 API，但是大多数客户端只需要初始化 HTTP 处理然后通过 web 页面访问数据。为了只利用包的潜在作用而引入包，可以将包名重命名为空标识符：

```go
import _ "net/http/pprof"
```

这种引入形式清楚的表明了这个包是为了他的潜在作用而引入，因为我们已经没有别的使用这种包的可能性：在这个文件里，他甚至连名称都没有。（如果他有名称，而且我们未使用的话，编译器会拒绝这段源码。）

## 接口检查

就像我们之前在讨论[接口](#接口)时看到的，一个类型无需显示的声明他实现了某种接口。反之，一个类型只需要实现了该接口需要的方法，则他就默认实现了这个接口。在实践中，大部分的接口转换是静态的，因此他们会在编译时被检查。例如，在一个需要 io.Reader 参数的函数中传入 *os.File 不会被编译，除非 *os.File 实现了 io.Reader 接口。

尽管如此，有些接口检查确实是在运行时发生的。一个例子是 [encoding/json](https://go.dev/pkg/encoding/json/) 包，其中定义了 [Marshaler](https://go.dev/pkg/encoding/json/#Marshaler) 接口。当 JSON 编码器接收到实现了该接口的值时，编码器使用该值的编码方法将其转换为 JSON，否则使用基础方法转换。编码器在运行时使用[类型断言](#接口转换与类型断言)检查此属性：

```go
m, ok := val.(json.Marshaler)
```

如果仅仅只需要了解类型是否实现了某个接口，但并不使用接口值自身，也许就是作为错误检查的一部分，可以使用空标识符来忽略类型断言的值：

```go
if _, ok := val.(json.Marshaler); ok {
    fmt.Printf("value %v of type %T implements json.Marshaler\n", val, val)
}
```

这个情况出现的一个地方是在实现类型的包中需要保证它实际上满足了接口。如果一个类型——例如，[json.RawMessage](https://go.dev/pkg/encoding/json/#RawMessage)——需要一个定制化的 JSON 表示，那么他应该实现 json.Marshaler 接口，但是这里没有静态类型转换使得编译器去自动检查这一点。如果这个类型的不经意间的改动无法满足了接口要求，JSON 编码器仍然会工作，但是不再使用定制化的实现方式。为了确保可以采用正确的实现，可以在这个包中进行一个全局的使用空标识符的声明：

```go
var _ json.Marshaler = (*RawMessage)(nil)
```

在这个声明中，赋值语句调用了一个从 *RawMessage 到 Marshaler 的类型转换，这需要 *RawMassage 实现 Marshaler 接口，且这个属性会在编译时被检查。如果 json.Marshaler 接口发生变化，这个包将不再编译，我们会注意到这一点，意识到这个包需要被更新。

这里的空标识符表示这个声明的存在仅仅是为了做类型检查，而非创建一个变量。尽管如此，不要对每个实现接口的类型做这种检查。为了方便起见，这种声明只在代码中没有静态转换的使用使用，这其实是一种很罕见的情况。

# 嵌入式

Go 没有提供传统的、类型驱动的子类概念，但他拥有通过 *嵌入类型* 来借用结构体或接口部分实现的能力。

接口嵌入十分简单，我们之前已经提过 io.Reader 和 io.Writer 接口；这里是他们的定义。

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

io 包还提供了其他几个接口，用来指定对象需要满足这些方法。例如，io.ReadWriter 是一个同时包括了读写的接口。我们可以通过显示的列举这两个方法的方式来指定 io.ReadWriter 接口，但是更简单且优雅的方式是直接将原先的两个接口嵌入其中，就像这样：

```go
// ReadWriter is the interface that combines the Reader and Writer interfaces.
type ReadWriter interface {
    Reader
    Writer
}
```

他的作用不言自明：ReadWriter 等同于 Reader 加上 Writer；他的能力由被嵌入接口组合而成。只有接口可以被接口嵌入。

这个想法同样适用于结构体，但是实现的更加深入。bufio 包有两个结构体类型，bufio.Reader 和 bufio.Writer，他们当然各自实现了 io 包中的接口。同时，bufio 包也实现了一个基于 buffer 的读写结构体，他通过嵌入的方式将 reader 和 writer 结合到一起：他包括了结构体内部的类型，但没有给他们字段名。

```go
// ReadWriter stores pointers to a Reader and a Writer.
// It implements io.ReadWriter.
type ReadWriter struct {
    *Reader  // *bufio.Reader
    *Writer  // *bufio.Writer
}
```

被嵌入的元素是已经正确初始化的结构体的指针。ReadWriter 结构体可以被写为

```go
type ReadWriter struct {
    reader *Reader
    writer *Writer
}
```

但随后为了暴露字段的方法，且需要满足接口的需求，我们还需要提供对方法的转发，就像这样：

```go
func (rw *ReadWriter) Read(p []byte) (n int, err error) {
    return rw.reader.Read(p)
}
```

通过直接嵌入结构体，我们可以避免这样的抄书工作。被嵌入类型的方法可以直接调用，这意味着 bufio.ReadWriter 不仅有 bufio.Reader 和 bufio.Writer 的方法，他其实同时满足了三个接口：io.Reader，io.Writer，io.ReadWriter。

嵌入和子类有一个很重要的区别。当我们嵌入一个类型时，该类型的方法会变成外层的方法，但是当方法被调用时，方法的接收者是内部类型，而非外部类型。在我们的例子中，当 bufio.ReadWriter 中的 Read 方法被调用时，他实际上的效果和我们将这个方法转发到外层相同；方法的接收者是 ReadWriter 中的 reader 字段，而非 ReadWriter 自身。

嵌入式也带来一些简单的小技巧。这个例子里我们同时使用了一个嵌入式字段和一个常规的命名字段。

```go
type Job struct {
    Command string
    *log.Logger
}
```

现在 Job 类型就拥有了 Print， Printf， Println 和其他 *log.Logger 带来的方法。我们当然可以选择给 Logger 提供一个字段名，但这没有必要。现在，当初始化完成后，我们可以这样输出 Job：

```go
job.Println("starting now...")
```

Logger 也是 Job 结构体中的一个字段，因此我们可以采用普通的方式在 Job 的构造函数中初始化他，例如这样，

```go
func NewJob(command string, logger *log.Logger) *Job {
    return &Job{command, logger}
}
```

或者采用复合字面量，

```go
job := &Job{command, log.New(os.Stderr, "Job: ", log.Ldate)}
```

如果我们需要直接引用被嵌入的字段，可以将他的类型名称（忽略包名）直接视作字段名称，就像我们在 ReadWriter 结构体中的 Read 方法中做的那样。在这里，如果我们需要访问 Job 类变量 job 中的 *log.Logger，我们可以使用 job.Logger，这对于如果我们想要重写 Logger 中的方法时很有用。

```go
func (job *Job) Printf(format string, args ...interface{}) {
    job.Logger.Printf("%q: %s", job.Command, fmt.Sprintf(format, args...))
}
```

嵌入类型引入了名称冲突的问题，但解决他们的规则很简单。首先，一个名为 X 的字段或方法会将其他更深层的的名为 X 的部分隐藏。如果 log.Logger 包含一个名为 Command 的字段或方法，那么 Job 中的 Command 方法会屏蔽他。

其次，如果相同层级中出现了相同的名称，那通常会是一个错误；如果 Job 结构体包含另一个名为 Logger 的字段或方法，则嵌入 log.Logger 是错误的。然而，如果在类型定义外的任何地方，程序都没有使用这个重复的名称，那么是没有问题的。这种限定可以防止外部嵌入的类型发生更改时带来的一些问题；如果从未使用过某个字段，即使它与另一个子类型中的字段冲突也无关紧要。

# 并发

## 通过通信共享

并发编程是一个庞大的主题，这里仅仅介绍一些 Go 相关的重点内容。

由于实现对共享变量的访问有很多微妙的细节，这导致了在很多环境中并发编程十分困难。Go 鼓励采用另一种方法，即通过通信传递共享的值，而非使用多个分离的线程来访问他。在任何时候只会有一个 goroutine 在使用这个值。通过这种设计方式，根本不会产生数据竞态问题。为了鼓励这种思维方式我们把他提炼成了一句简单的口号：

> 不要通过共享内存来通信，而是通过通信来共享内存。

这种设计可能会矫枉过正。例如，对于一个引用计数器，最好的方法就是在整数上加一个 mutex。但是作为一种高级方法，使用管道来控制接入还是会让编写清晰、正确的程序更加简单。

一种理解这个模型的方法是，考虑一个在 CPU 上运行的典型的单线程程序。他不需要任何的同步关键字。现在运行该程序的另一个实例，他也不需要任何同步关键字。现在让他们之间进行通信，如果通信过程是同步的，那么就仍然不需要任何同步关键字。Unix 中的管道就是这个模型的完美实例。尽管 Go 的并发方法起源于 Hoare 的 通信顺序进程（CSP），但他也可以被看作一种类型安全的 Unix 管道。

## Goroutines

*goroutines* 被这样命名是因为所有现有的术语：线程、协程、进程等等，都无法准确表达他的内涵。goroutine 有一个简单的模型：他是一个与其他 goroutines 在同一地址空间并发执行的函数。他非常的轻量级，开销仅仅略多于堆栈空间分配。开始时仅使用一个小堆栈，因此开销很低，随后在使用时按序分配（或释放）堆存储。

Goroutines 在操作系统的多个线程上多路复用，如果其中一个发生堵塞，例如在等待 I/O，其他的 goroutines 可以继续运行。这种设计隐藏了很多线程创建和管理上的复杂性。

在函数或方法调用前添加 go 关键字会让这次调用运行在一个新建的 goroutine 中。当调用完成后，goroutine 静默的退出。（效果类似于在 Unix shell 中使用 & 在后台运行命令。）

```go
go list.Sort()  // run list.Sort concurrently; don't wait for it.
```

匿名函数可以方便的通过 goroutine 调用。

```go
func Announce(message string, delay time.Duration) {
    go func() {
        time.Sleep(delay)
        fmt.Println(message)
    }()  // Note the parentheses - must call the function.
}
```

在 Go 中，匿名函数是闭包：实现确保了函数引用的变量的生命周期至少和函数一致。

这些例子都不是很典型，因为函数无法发出完成信号。对此，我们需要 channels。

## Channels

类似于 maps，channels 同样使用 make 进行分配，获得的值同样引用了一个底层的数据结构。分配 channels 的 make 提供了一个可选的整数参数，用于设定 channel 的缓冲大小。默认值是 0，表示一个无缓冲的同步通道。

```go
ci := make(chan int)            // unbuffered channel of integers
cj := make(chan int, 0)         // unbuffered channel of integers
cs := make(chan *os.File, 100)  // buffered channel of pointers to Files
```

无缓冲的 channels 结合了通信（值的交换）与同步（保证两个计算(goroutines)）处在一个已知的状态。

关于通道的使用有很多好的惯例，我们从下面这个开始。在这段代码中我们在后台启动了数组排序。而 channel 允许启动者的 goroutine 等待排序完成。

```go
c := make(chan int)  // Allocate a channel.
// Start the sort in a goroutine; when it completes, signal on the channel.
go func() {
    list.Sort()
    c <- 1  // Send a signal; value does not matter.
}()
doSomethingForAWhile()
<-c   // Wait for sort to finish; discard sent value.
```

接收者会在收到数据之前一直阻塞。如果 channel 是无缓冲的，发送者也会在接收者接受数据前一直堵塞。如果通道是有缓冲的，发送者只会在向缓冲区满的通道再次发送数据的时候堵塞，这意味着需要等待一些接收者来领取数据。

有缓冲的 channel 可以像信号量一样使用，例如用来进行吞吐量的限制。在这个例子中，输入的请求被传递到 handle，而他将一个值放入 channel、处理请求、最后从 channel 中接收一个值，为下一个使用者准备好“信号量”。channel 缓冲器的容量限制了 process 函数的并发数量。

```go
var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
    sem <- 1    // Wait for active queue to drain.
    process(r)  // May take a long time.
    <-sem       // Done; enable next request to run.
}

func Serve(queue chan *Request) {
    for {
        req := <-queue
        go handle(req)  // Don't wait for handle to finish.
    }
}
```

一旦有 MaxOutstanding 数量的 handlers 同时在处理，更多的请求会由于尝试向满载的缓冲 channel 写入数据而被堵塞，直到其中某个现存的 handlers 完成计算并且从缓冲 channel 中接收数据。

不过，这样的设计仍然存在一个问题：Serve 对每一个进入的请求创建新的 goroutine，即便在任何时候只有 MaxOutstanding 个请求可以运行。这样带来的结果是，当请求到来的过快时，该程序可能会消耗大量的资源。我们可以将 goroutines 的创建移入 Serve 来解决这个问题。这里有一个很明显的解决方案，但是小心，现在这里有一个 bug，我们随后会修复：

```go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func() {
            process(req) // Buggy; see explanation below.
            <-sem
        }()
    }
}
```

这里的 bug 是，在 Go 循环中，循环变量是在每次迭代时共享的，因此 req 变量是在所有 goroutines 中共享的，这并不符合我们的预期，我们希望每个 goroutine 中的 req 是独立的。这里有一个解决方式，将 req 的值作为参数传递给 goroutines 的闭包：

```go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func(req *Request) {
            process(req)
            <-sem
        }(req)
    }
}
```

对比这个版本和上一个版本的代码，可以观察闭包声明和运行中的区别。另一个解决方案是创建一个同名变量，比如在这个例子：

```go
func Serve(queue chan *Request) {
    for req := range queue {
        req := req // Create new instance of req for the goroutine.
        sem <- 1
        go func() {
            process(req)
            <-sem
        }()
    }
}
```

这个写法也许看起来很怪

```go
req := req
```

但这是合法的，而且很符合 Go 中的习惯。这会产生一个同名的新变量，有意的在循环体中隐藏了循环变量，确保了每个 goroutine 中变量的唯一性。

回到编写这个服务器的问题，另一个可以良好管理资源的解决方案是启动固定数量的 goroutines 并使他们都去读取 request channel。goroutines 的数量限制了 process 并发调用的数量。Serve 函数同时也接受一个通知其退出的 channel，在启动 goroutines 之后他阻塞直到从该通道接收到内容。

```go
func handle(queue chan *Request) {
    for r := range queue {
        process(r)
    }
}

func Serve(clientRequests chan *Request, quit chan bool) {
    // Start handlers
    for i := 0; i < MaxOutstanding; i++ {
        go handle(clientRequests)
    }
    <-quit  // Wait to be told to exit.
}
```

## Channels of channels

Go 中最重要的特性之一就是 channel 是 Go 中的一等公民，他可以像其他值一样被分配和传递。这个属性的常见用途之一是用来实现安全、并行的解复用。

在上一章节的例子中，handle 是一个想象中用来处理请求的模型，但是我们没有定义他所处理的具体类型。如果该类型包含接收回复的通道，那么每个客户端都可以独立定义他们接收计算结果的路径。这里是一个对 Request 类型定义的示意。

```go
type Request struct {
    args        []int
    f           func([]int) int
    resultChan  chan int
}
```

客户端提供了计算函数，计算参数，以及在其内部的用来接收结果的 channel。

```go
func sum(a []int) (s int) {
    for _, v := range a {
        s += v
    }
    return
}

request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
// Send request
clientRequests <- request
// Wait for response.
fmt.Printf("answer: %d\n", <-request.resultChan)
```

而在服务端一侧，唯一的改变就是 handler 函数的内容。

```go
func handle(queue chan *Request) {
    for req := range queue {
        req.resultChan <- req.f(req.args)
    }
}
```

显然还需要很多代码工作才能让这个例子成为真正的实现，但是这些代码可以视作一个有限速的、并发的、非阻塞的 RPC 系统的框架，而且看不到任何一个 mutex。

## 并行处理

关于这些想法的另一个应用是在多个 CPU 核心上并行处理计算任务。如果计算任务可以被分解成多个独立执行的小块，那么他就可以被并行处理，只需为每一块分配一个 channel 来标志其完成。

假设我们有一个开销很大的向量计算操作，而且对每个元素的计算是独立的，就像这个理想化的例子。

```go
type Vector []float64

// Apply the operation to v[i], v[i+1] ... up to v[n-1].
func (v Vector) DoSome(i, n int, u Vector, c chan int) {
    for ; i < n; i++ {
        v[i] += u.Op(v[i])
    }
    c <- 1    // signal that this piece is done
}
```

我们在循环中按照 CPU 的数量独立启动每个计算任务。他们可能会按照任意顺序完成但是这不重要，我们只需要在启动所有 goroutines 后通过清空 channel 来对完成信号进行计数。

```go
const numCPU = 4 // number of CPU cores

func (v Vector) DoAll(u Vector) {
    c := make(chan int, numCPU)  // Buffering optional but sensible.
    for i := 0; i < numCPU; i++ {
        go v.DoSome(i*len(v)/numCPU, (i+1)*len(v)/numCPU, u, c)
    }
    // Drain the channel.
    for i := 0; i < numCPU; i++ {
        <-c    // wait for one task to complete
    }
    // All done.
}
```

相比于创建一个固定值的 numCPU，我们可以在运行时获得一个更合适的值。函数 [runtime.NumCPU](https://go.dev/pkg/runtime#NumCPU) 返回运行机器的 CPU 硬件核心数量，因此我们可以

```go
var numCPU = runtime.NumCPU()
```

还有一个函数是 [runtime.GOMAXPROCS](https://go.dev/pkg/runtime#GOMAXPROCS)，他可以报告（或设置）由用户定义的 Go 程序运行时可以使用的核心数。他的默认值是 runtime.NumCPU，但是可以被一个名称相似的环境变量修改，或是被调用这个函数并传入一个正整数修改。调用此函数并传入 0 会查询这个值。因此，如果我们想尊重用户设置的资源限制，我们可以

```go
var numCPU = runtime.GOMAXPROCS(0)
```

请注意不要混淆并发（将程序结构化为独立的执行组件）和并行（在多个 CPU 上并行计算以提高效率）的概念。虽然 Go 语言的并发特性可以让一些问题易于结构化为并行计算，但 Go 语言是一种并发语言，而不是并行语言，并不是所有并行化问题都适合 Go 语言的模型。关于这两种概念的区别，可以参考这篇[博客](https://go.dev/blog/concurrency-is-not-parallelism)中引用的演讲。

## A leaky buffer

并发编程的工具甚至可以让非并发的想法更容易表达。这里有一个对某个 RPC 包的抽象例子。客户端 goroutine 从某个数据源循环的接收数据，也许是通过网络。为了避免大量分配和释放 buffer 的开销，他持有了一个空闲 buffer 的列表，然后通过一个带缓冲区的 channel 来表示这个列表。如果 channel 是空的，那么就分配一个新的 buffer。一旦 buffer 的内容被填充完成，他使用 serverChan 将其发送至服务程序。

```go
var freeList = make(chan *Buffer, 100)
var serverChan = make(chan *Buffer)

func client() {
    for {
        var b *Buffer
        // Grab a buffer if available; allocate if not.
        select {
        case b = <-freeList:
            // Got one; nothing more to do.
        default:
            // None free, so allocate a new one.
            b = new(Buffer)
        }
        load(b)              // Read next message from the net.
        serverChan <- b      // Send to server.
    }
}
```

server 循环接收来自客户端的消息，处理，并将 buffer 返回到空闲列表。

```go
func server() {
    for {
        b := <-serverChan    // Wait for work.
        process(b)
        // Reuse buffer if there's room.
        select {
        case freeList <- b:
            // Buffer on free list; nothing more to do.
        default:
            // Free list full, just carry on.
        }
    }
}
```

客户端尝试从空闲列表中取出一个 buffer，如果没有空闲的 buffer ，那么他会分配一个新的 buffer。服务器将 b 发送到空闲列表除非 freeList 已满，这时 buffer 会被丢弃之后被垃圾回收器回收。（select 语句中的 default 条件会在其他 case 都不生效时被选中，这意味着这条 select 语句永远不会堵塞。）在带缓冲区的 channel 和垃圾回收机制的共同作用下，这个实现仅使用了几行代码就构建了一个泄露桶式的 buffer 池。

# 错误

库例程必须经常向调用者返回一些错误信息。就像我们之前提过的，Go 中的多重返回让我们可以轻松的在正常的返回值旁边附带详细的错误描述。使用这个特性来提供详细的错误信息是很好的风格。例如，就像我们即将看到的，os.Open 不仅仅是在故障时返回一个空指针，他还提供了一个错误值来表述错误的内容。

为了方便起见，错误被定义为类型 error，一个简单的内建接口。

```go
type error interface {
    Error() string
}
```

库的作者可以在这个包装下自由的使用更丰富的模型实现该接口，让错误信息不仅包括错误本身，同时提供一些其他的内容。就像之前提过的，除了返回的 *os.File 类型的值之外，os.Open 还返回一个 error 类型的值。如果文件被成功的打开，那么这个 error 值将会是 nil，但是如果发生了错误，他将会持有一个 os.PathError：

```go
// PathError records an error and the operation and
// file path that caused it.
type PathError struct {
    Op string    // "open", "unlink", etc.
    Path string  // The associated file.
    Err error    // Returned by the system call.
}

func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}
```

PathError 的 Error 方法生成的字符串类似于这样：

```text
open /etc/passwx: no such file or directory
```

像这样一个 error，囊括了有问题的文件名称、操作内容和他触发了的系统错误，即使错误打印的地方和调用处相距甚远，这样的报错依然十分有用；他比直接的打印 “no such file or directory” 能提供更多有效的信息。

如果有条件的话，error 字符串应该标注他们的来源，例如添加前缀来标注是哪个操作或者包生成了这个 error。例如，在 image 包中，因为未知格式而导致的解码错误的错误字符串为 “image: unknown format”。

关心精确的错误细节的调用者可以使用 type switch 或者类型断言来获得 error 的具体类型从而获取更多的细节。对于 PathErrors 来说，这可能包括了检查内部的 Err 字段用来处理可恢复的故障。

```go
for try := 0; try < 2; try++ {
    file, err = os.Create(filename)
    if err == nil {
        return
    }
    if e, ok := err.(*os.PathError); ok && e.Err == syscall.ENOSPC {
        deleteTempFiles()  // Recover some space.
        continue
    }
    return
}
```

这里的第二个 if 语句是一个[类型断言](接口转换与类型断言)。如果他失败了，ok 的值会是 false，e 的值会是 nil。如果他成功了，ok 的值会是 true，这表示 error 的类型确实是 *os.PathError，此时 e 会成为这个类型，这样我们可以检查其中的更多信息细节。

## Panic

常规的向调用者汇报错误的方式是添加一个 error 类型的返回值。Read 方法就是一个广为人知的典范，他返回了读取的字节数和一个 error。但是，当发生了不可恢复的故障的情况下怎么办呢？有些情况下就是需要中断程序运行。

为了实现这个目的，有一个内建函数 panic，他可以创建一个运行时错误从而中止程序（但也有例外，参考下一章）。此函数需要一个任意类型的信号参数——通常是字符串——用作程序终止时的打印输出。他也可以用来表明发生了某些不该发生的事，例如存在一个无限循环。

```go
// A toy implementation of cube root using Newton's method.
func CubeRoot(x float64) float64 {
    z := x/3   // Arbitrary initial value
    for i := 0; i < 1e6; i++ {
        prevz := z
        z -= (z*z*z-x) / (3*z*z)
        if veryClose(z, prevz) {
            return z
        }
    }
    // A million iterations has not converged; something is wrong.
    panic(fmt.Sprintf("CubeRoot(%g) did not converge", x))
}
```

这里只是一个例子，实际上的库函数应该避免 panic。如果问题可以被掩盖或者解决，让程序继续运行下去总比直接中断整个程序要好。一个可能的反例是初始化：可以认为，如果一个库确实无法完成自身设置，那么进行 panic 也是合理的。

```go
var user = os.Getenv("USER")

func init() {
    if user == "" {
        panic("no value for $USER")
    }
}
```

## Recover

当 panic 发生时，或者包括隐式的运行时错误例如切片越界、类型断言错误等等，他会立刻中断当前执行的函数并且开始释放当前 goroutine 的堆栈，运行沿途上的任何 defer 函数。如果这个释放过程到达了当前 goroutine 的顶部，程序则会终止。然而，可以使用内建函数 recover 来恢复对 goroutine 的控制并让其继续执行。

对 recover 的调用会中止释放过程，并返回传递给 panic 的参数。由于在释放过程中唯一能够被执行的代码是位于 defer 中的代码，因此 recover 只有在 defer 函数中才有作用。

recover 的一个应用是中断服务中失败的 goroutine，从而避免中断整个程序。

```go
func server(workChan <-chan *Work) {
    for work := range workChan {
        go safelyDo(work)
    }
}

func safelyDo(work *Work) {
    defer func() {
        if err := recover(); err != nil {
            log.Println("work failed:", err)
        }
    }()
    do(work)
}
```

在这个例子中，如果 do(work) 导致了 panic，错误结果将被日志记录，而且报错的 goroutine 只会干净的退出而不会影响其他协程。无需再 defer 闭包中再添加任何东西，调用 cover 就完全足够。

除非是直接被 defer 函数调用，否则 recover 都会返回 nil，因此 defer 中的代码可以调用那些本身也使用了 panic 和 recover 的库函数。例如，在 safelyDo 函数中的 defer 函数可以在 recover 之前调用 logging 函数，该函数的执行不会受到 panic 过程的影响。

理解了 recover 的工作模式后，我们可以通过调用 panic 让 do 函数（以及他调用的任何函数）在遇到错误情况时干净的退出。我们可以利用这个概念来简化复杂软件中的错误处理。让我们看看这个理性化的 regexp 包，他通过调用一个带有本地定义错误类型的 panic 来报告解析错误。这里是关于 Error、error 方法和 Compile 函数的定义。

```go
// Error is the type of a parse error; it satisfies the error interface.
type Error string
func (e Error) Error() string {
    return string(e)
}

// error is a method of *Regexp that reports parsing errors by
// panicking with an Error.
func (regexp *Regexp) error(err string) {
    panic(Error(err))
}

// Compile returns a parsed representation of the regular expression.
func Compile(str string) (regexp *Regexp, err error) {
    regexp = new(Regexp)
    // doParse will panic if there is a parse error.
    defer func() {
        if e := recover(); e != nil {
            regexp = nil    // Clear return value.
            err = e.(Error) // Will re-panic if not a parse error.
        }
    }()
    return regexp.doParse(str), nil
}
```

如果 doParse 触发了 panic，recover 代码块会将返回值设置为nil——因为 defer 函数可以修改命名返回值的值。之后，他会在向 err 赋值的语句中通过将 e 类型断言为本地类型 Error，检查问题是否是解析错误。如果不是，那么类型断言会失败，引发一个运行时错误，从而导致堆栈继续释放，就像释放过程未曾中断这样。这种检查意味着当有一些类似于数组越界的运行时错误发生时，即使我们使用 panic 和 recover 处理了解析错误，代码仍然会失败。

在错误处理机制到位后，error 方法（虽然他和内建类型 error 同名，但是由于他是一个绑定在类型上的方法，因此这个命名没有问题，使用起来也很自然）使得我们可以更简单的上报解析错误，而无需操心于手动释放解析堆栈：

```go
if pos == 0 {
    re.error("'*' illegal at start of expression")
}
```

虽然这个模式非常实用，但是他应该仅限在包内部使用。通过解析将内部的 panic 转化为 error 类型的值，而不是将 panic 暴露给自己的客户端，这是一个良好的规则。

顺带一提，这种类型的用法改变了实际报错中的 panic 内容。然而，不管是原始的还是新的报错内容都会在崩溃日志中体现，因此报错的根源依然可以被找到。因此这种简单的 re-panic 机制往往就够用了——毕竟最终都是要崩溃的——但是如果你只想在崩溃日志中展示原始错误，你可以通过写一小段代码来过滤非预期的错误然后使用原始的错误来 re-panic。这是留给读者的小练习。

# 一个WEB服务器

让我们用一个完整的 Go 程序来收尾，一个 WEB 服务器。这其实是一个基于其他服务的 web 服务器。Google 在 chart.apis.google.com 上提供了一个服务，可以自动将数据格式化为图表和图形。但是他的交互比较麻烦，因为你要把数据作为 URL 的一部分用来查询。这个程序提供了一个更好的数据格式接口：通过输入一段文本，他调用图表服务来生成一个二维码，他可以被您的手机扫描并且转换为一个 URL，这样您就无需在手机的小键盘上输入这个 URL。

这里是完整的程序，随后会有说明。

```go
package main

import (
    "flag"
    "html/template"
    "log"
    "net/http"
)

var addr = flag.String("addr", ":1718", "http service address") // Q=17, R=18

var templ = template.Must(template.New("qr").Parse(templateStr))

func main() {
    flag.Parse()
    http.Handle("/", http.HandlerFunc(QR))
    err := http.ListenAndServe(*addr, nil)
    if err != nil {
        log.Fatal("ListenAndServe:", err)
    }
}

func QR(w http.ResponseWriter, req *http.Request) {
    templ.Execute(w, req.FormValue("s"))
}

const templateStr = `
<html>
<head>
<title>QR Link Generator</title>
</head>
<body>
{{if .}}
<img src="http://chart.apis.google.com/chart?chs=300x300&cht=qr&choe=UTF-8&chl={{.}}" />
<br>
{{.}}
<br>
<br>
{{end}}
<form action="/" name=f method="GET">
    <input maxLength=1024 size=70 name=s value="" title="Text to QR Encode">
    <input type=submit value="Show QR" name=qr>
</form>
</body>
</html>
`
```

main 之前的部分应该都很容易理解。flag 为我们的服务设置了一个默认的 HTTP 端口。模板变量 templ 是有趣的地方。他构建了一个服务器用来显示页面的 HTML 模板，稍后会详细介绍。

main 函数中解析了 flags，使用我们之前讨论过的机制将 QR 函数绑定在服务的根路径。之后 http.ListenAndServe 被调用，启动 HTTP 服务，并且在服务运行期间阻塞主程序。

QR 接受请求包含表单数据的请求，根据表单中名为 s 的数据的值来执行模板。

模板包 html/template 非常强力，这个程序仅仅涉及他功能的一小部分。本质上，它通过替换从传递给 templ.Execute 的数据项派生的元素来动态重写一段 HTML 文本，在这里例子中就是表单的值。在模板文本 (template Str) 中，双括号分隔的部分表示模板操作。仅当数据项 .（点） 非空时，{%raw%}{{if .}}{%endraw%} 到 {%raw%}{{end}}{%endraw%} 之间的代码片才会执行。这意味着，当传入字符串为空时，这一部分的模板将不生效。

两个 {%raw%}{{.}}{%endraw%} 表示了两个提供给模板的数据——一个位于查询字符串——另一个直接在 web 页面中。HTML 模板包自动提供适当的转移，以便文本可以安全的显示。

模板中剩余部分的字符串只是当页面加载时显示的 HTML。如果这部分的解释过于简短，您也可以参考这篇关于模板包的[文档](#https://go.dev/pkg/html/template/)进行更全面的讨论。

此时你已经拥有了一个仅使用几行代码创建的数据驱动的 HTML 服务。Go 就是这样，他有能力仅仅使用几行代码来实现很多事情。
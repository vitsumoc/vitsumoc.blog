---
title: "[翻译]Go教程：泛型入门"
url: "Go教程：泛型入门"
date: 2024-04-12 10:42:27
categories:
- Go
tags:
- Go
- 翻译
---

> 原文 [Tutorial: Getting started with generics](https://go.dev/doc/tutorial/generics)

<!-- more -->

<head>
  <style>
    .indentation {
      margin-left: 2rem;
    }
  </style>
</head>

这篇教程介绍了 Go 中泛型的基础。通过泛型，您可以定义和使用一些函数或类型，这些函数或类型可以用于处理调用代码提供的任意类型集合。

在这个教程中，您将声明两个简单的非泛型函数，随后在一个单一的泛型函数中实现同样的逻辑。

您将按照如下步骤进行：

1. 为您的代码创建文件夹。
2. 一个非泛型的函数。
3. 添加一个可以处理多种类型的泛型函数。
4. 在调用泛型函数时删除类型参数。
5. 声明一个类型约束。

**备注：**其他教程请查看[链接](https://go.dev/doc/tutorial/index.html)。

**备注：**如果您需要的话，您也可以使用 [“Go dev branch” 模式的 Go playground](https://go.dev/play/?v=gotip) 来编辑和运行您的程序。

# 准备工作

- **Go 1.18 及以上。**有关安装说明，请参考 [Installing Go](https://go.dev/doc/install)。
- **代码编辑工具。**任何文本编辑器均可。
- **命令行终端。**Go 可以在 Linux 和 Mac 的终端中运行，也可以在 Windows 的 PowerShell 或 cmd 中运行。

# 创建文件夹

为了开始一切，先创建文件夹用来存放您即将编写的代码。

1. 打开一个命令提示符并切换到您的家目录。

<div class="indentation">

在 Linux 或者 Mac 上：

```bash
$ cd
```

在 Windows 上：

```bat
C:\> cd %HOMEPATH%
```

教程的剩余部分我们都会将 $ 作为提示符。您使用的命令在 Windows 中也会生效。

</div>

2. 在命令提示符中，为您的代码创建名为 generics 的文件夹。

<div class="indentation">

```bash
$ mkdir generics
$ cd generics
```

</div>

3. 为您的代码创建一个模块。

<div class="indentation">

运行 `go mod init` 命令，为他提供您的新代码的模块路径。

```bash
$ go mod init example/generics
go: creating new go.mod: module example/generics
```

**备注：**对于用于生产的代码，您最好根据您的需要去选择模块路径。有关更多消息，请参考<a href="https://go.dev/doc/modules/managing-dependencies">Managing dependencies</a>。

</div>

接下来，您会添加一些处理 map 的简单代码。

# 非泛型函数

在这一步中，您会创建两个函数，他们都会将 map 中的值加总，并返回结果。

您声明两个函数，而非一个，因为您需要处理两种不同类型的 map：一个存储 int64 类型的值，另一个存储 float64 类型的值。

## 编写代码

1. 使用您的文本编辑器，在 generics 文件夹中创建一个名为 main.go 文件。您将要在这里编写您的 Go 代码。

2. 打开 main.go，粘贴如下包声明。

<div class="indentation">

```go
package main
```

独立的程序（而非库）始终使用 main 包。

</div>

3. 在包声明下方，粘贴如下两个函数声明。

<div class="indentation">

```go
// SumInts adds together the values of m.
func SumInts(m map[string]int64) int64 {
    var s int64
    for _, v := range m {
        s += v
    }
    return s
}

// SumFloats adds together the values of m.
func SumFloats(m map[string]float64) float64 {
    var s float64
    for _, v := range m {
        s += v
    }
    return s
}
```

在上述代码中，您完成了：

- 声明了两个将 map 中的值加总并返回的函数。
  - SumFloats 接收类型为 \[string\]float64 的 map。
  - SumInts 接收类型为 \[string\]int64 的 map。

</div>

4. 在 main.go 文件的顶部，包声明的下方，粘贴 main 函数，他初始化两个 map，并将他们当作您之前声明的函数的参数来使用。

<div class="indentation">

```go
func main() {
    // Initialize a map for the integer values
    ints := map[string]int64{
        "first":  34,
        "second": 12,
    }

    // Initialize a map for the float values
    floats := map[string]float64{
        "first":  35.98,
        "second": 26.99,
    }

    fmt.Printf("Non-Generic Sums: %v and %v\n",
        SumInts(ints),
        SumFloats(floats))
}
```

在上述代码中，您：

- 初始化了 float64 值的 map 和 int64 值的 map，每个都有两条数据。
- 调用之前定义的两个函数，查询两个 map 各自的总值。
- 打印结果

</div>

5. 在 main.go 的顶部附近，在包声明的下方，引入您代码中需要的包。

<div class="indentation">

将文件顶部修改为：

```go
package main

import "fmt"
```

</div>

6. 保存 main.go。

## 运行代码

在命令行中，main.go 文件所在的文件夹路径下，运行命令。

```bash
$ go run .
Non-Generic Sums: 46 and 62.97
```

通过泛型，您可以只编写一个函数，而非两个。接下来，您会添加一个单独的泛型函数，他可以用于 int64 类型的 map 或 float64 类型的 map。

# 泛型函数

在这一节中，您将要添加一个泛型函数，他可以接收整数或者浮点数类型的 map 作为参数，他可以用来替代您之前编写的两个函数。

为了同时支持两种类型的值，这个函数需要通过一种方式来声明他支持的类型。从另一方面来说，调用代码需要有一种方式来指定他使用何种类型的 map 来调用该函数。

为了实现这个目标，您声明的函数除了有普通的函数参数外，还需要有 *类型参数（type parameters）*。正是这些类型参数支撑了泛型函数，让他们能够应付不同类型的参数。您在调用泛型函数时需要同时传入类型参数和函数参数。

每个类型参数都有一个 *类型约束（type constraint）* 来扮演他的元类型。每个类型约束指定了调用函数在传入各类型参数时允许使用的类型参数范围。

虽然类型参数约束往往提供了多种类型，但在编译时类型参数只代表一种类型——调用代码所提供的类型。如果调用代码提供的类型不满足类型参数约束，那么编译就会失败。

请记住，类型参数必须支持泛型代码中的任何操作。例如，如果您泛型函数中的代码包含字符串的操作（例如取索引），但您的类型参数约束包括了数字类型，那么编译就会失败。

在您接下来要编写的代码中，您将实现供同时允许整数和浮点数的类型参数约束。

## 编写代码

1. 在您之前添加的两个函数下方，粘贴如下泛型函数。

<div class="indentation">

```go
// SumIntsOrFloats sums the values of map m. It supports both int64 and float64
// as types for map values.
func SumIntsOrFloats[K comparable, V int64 | float64](m map[K]V) V {
    var s V
    for _, v := range m {
        s += v
    }
    return s
}
```

在这段代码中，您：

- 声明了带有两个类型参数（在中括号中）的 SumIntsOrFloats 函数，类型参数是 K 和 V，同时声明了一个使用类型参数定义的函数参数 m，他的类型是 map\[K\]V。函数返回值的类型也是 V。
- 将类型参数 K 的类型约束指定为可比较的（comparable）。Go 中的 comparable 约束就是专门为了这种情况定义的，他表示允许所有支持 == 和 != 的类型。Go 要求 map 的键必须是可比较的，因此您必须将 K 声明为 comparable，他才可以作为 map 中的 key。这也同时确保了调用代码需要使用合理的类型作为 map 的键。
- 将类型参数 V 的类型约束指定为两种类型，int64 和 float64。使用 | 分隔两种类型，表示这两种类型都可以被使用。在调用代码中使用任何一种都可以成功编译。
- 指定函数参数 m 的类型为 map\[K\]V，其中 K 和 V 是之前指定的类型参数。请注意，我们现在知道 map\[K\]V 是一个合法的 map，因为 K 必须是一个可比较类型。如果我们之前没有将 K 声明为可比较类型，编译器会拒绝引用 map\[K\]V。

</div>

2. 在 main.go 中，您已有代码的下方，粘贴如下代码。

<div class="indentation">

```go
fmt.Printf("Generic Sums: %v and %v\n",
    SumIntsOrFloats[string, int64](ints),
    SumIntsOrFloats[string, float64](floats))
```

在这段代码中，您：

- 调用了您之前申明的泛型函数，传入了您之前创建的 map。

- 指定类型参数——将类型名称放入方括号中——用来指明成为您当前调用的函数中的类型参数。

<div class="indentation">

正如您即将在下一小节中看到的，您往往可以省略调用函数时的类型参数。Go 可以通过您的代码推断他们。

</div>

- 打印函数的加总结果。

</div>

## 运行代码

在命令行中，main.go 文件所在的文件夹路径下，运行命令。

```bash
$ go run .
Non-Generic Sums: 46 and 62.97
Generic Sums: 46 and 62.97
```

为了运行您的代码，编译器会在每次调用时将类型参数替换为调用时设置的具体类型。

在调用您的泛型函数的过程中，编译器会使用您指定的类型参数替换函数中的类型参数。正如您即将在下一小节中看到的，在很多情况下您可以省略类型参数，因为编译器会自动推断他们。

# 删除调用泛型函数时的类型参数

在这一小节，您将添加一个新版本的对泛型函数的调用，通过一些微小的改动来简化调用代码。您将会删除类型参数，实际上在这个例子中他们确实是不必要的。

当 Go 编译器可以推断您想要使用的类型时，您可以在调用代码中省略类型参数。编译器通过函数参数的类型来推断类型参数。

请记住，这并不总是可行的。例如，如果您希望调用一个没有函数参数的泛型函数，您就必须在调用时包含类型参数。

## 编写代码

- 在 main.go 中，您现有代码的下方，粘贴如下代码。

<div class="indentation">

```go
fmt.Printf("Generic Sums, type parameters inferred: %v and %v\n",
    SumIntsOrFloats(ints),
    SumIntsOrFloats(floats))
```

在这段代码中，您：

- 调用泛型函数，并省略了类型参数。

</div>

## 运行代码

在命令行中，main.go 文件所在的文件夹路径下，运行命令。

```bash
$ go run .
Non-Generic Sums: 46 and 62.97
Generic Sums: 46 and 62.97
Generic Sums, type parameters inferred: 46 and 62.97
```

接下来，您会通过将整数和浮点数设置为一种可复用的类型约束来进一步简化函数。

# 声明类型约束

在这个最后的小节中，您将要把您之前定义的约束做成一个接口，这样您就可以在更多地方复用他们。通过这种方式声明约束有助于精简代码，特别是当约束的内容逐渐复杂时。

您可以将类型约束声明为一个接口，这种约束允许任何类型来实现这个接口。例如，如果您声明了一个有三种方法的类型约束接口，并将他应用为一个泛型函数的类型参数，那么用于调用此函数的类型必须实现所有这些方法。

约束接口也可以指定特定的类型，就像您即将在本节中看到的那样。

## 编写代码

1. 在 main 函数之前，引入语句之后，粘贴下列用于声明类型约束的代码。

<div class="indentation">

```go
type Number interface {
    int64 | float64
}
```

在这段代码中，您：

- 声明了 Number 接口作为类型约束。

- 在 Number 接口中声明了 int64 和 float64。

<div class="indentation">

在本质上，您只是将函数中的类型声明换了个地方。此时，当您想要表示 int64 或 float64 的类型时，您可以直接使用 Number 类型，而非 int64 | float64。

</div>

2. 在您之前定义的函数下方，粘贴 SunNumbers 泛型函数。

```go
// SumNumbers sums the values of map m. It supports both integers
// and floats as map values.
func SumNumbers[K comparable, V Number](m map[K]V) V {
    var s V
    for _, v := range m {
        s += v
    }
    return s
}
```

在这段代码中，您：

- 声明了一个和原有泛型函数逻辑相同的泛型函数，但使用接口类型取代了原来的类型约束。就像之前一样，您使用类型参数作为函数参数和返回值的类型。

3. 在 main.go 中，您现有的代码下方，粘贴如下代码。

```go
fmt.Printf("Generic Sums with Constraint: %v and %v\n",
    SumNumbers(ints),
    SumNumbers(floats))
```

在这段代码中，您：

- 调用 SubNumbers，并用之前定义的 map 作为参数，打印每个加总的值。

<div class="indentation">

就像在前面章节中提到的，您在调用泛型函数时省略了类型参数（其中的类型需要被中括号包裹）。Go 编译器可以通过其他参数来推断类型参数。

</div>

</div>

## 运行代码

在命令行中，main.go 文件所在的文件夹路径下，运行命令。

```bash
$ go run .
Non-Generic Sums: 46 and 62.97
Generic Sums: 46 and 62.97
Generic Sums, type parameters inferred: 46 and 62.97
Generic Sums with Constraint: 46 and 62.97
```

# 结论

很好，您已经靠自己掌握了 Go 中的泛型。

向您推荐这些主题：

- [Go Tour](https://go.dev/tour/) 是一个非常优秀的对 Go 的介绍。
- 您可以在 [Effective Go](https://go.dev/doc/effective_go) 和 [How to write Go code](https://go.dev/doc/code) 中找到很多关于 Go 的最佳实践。

# 完整代码

您可以通过在 [Go playground](https://go.dev/play/p/apNmfVwogK0?v=gotip) 中点击 **Run** 按钮来运行这些代码。

```go
package main

import "fmt"

type Number interface {
    int64 | float64
}

func main() {
    // Initialize a map for the integer values
    ints := map[string]int64{
        "first": 34,
        "second": 12,
    }

    // Initialize a map for the float values
    floats := map[string]float64{
        "first": 35.98,
        "second": 26.99,
    }

    fmt.Printf("Non-Generic Sums: %v and %v\n",
        SumInts(ints),
        SumFloats(floats))

    fmt.Printf("Generic Sums: %v and %v\n",
        SumIntsOrFloats[string, int64](ints),
        SumIntsOrFloats[string, float64](floats))

    fmt.Printf("Generic Sums, type parameters inferred: %v and %v\n",
        SumIntsOrFloats(ints),
        SumIntsOrFloats(floats))

    fmt.Printf("Generic Sums with Constraint: %v and %v\n",
        SumNumbers(ints),
        SumNumbers(floats))
}

// SumInts adds together the values of m.
func SumInts(m map[string]int64) int64 {
    var s int64
    for _, v := range m {
        s += v
    }
    return s
}

// SumFloats adds together the values of m.
func SumFloats(m map[string]float64) float64 {
    var s float64
    for _, v := range m {
        s += v
    }
    return s
}

// SumIntsOrFloats sums the values of map m. It supports both floats and integers
// as map values.
func SumIntsOrFloats[K comparable, V int64 | float64](m map[K]V) V {
    var s V
    for _, v := range m {
        s += v
    }
    return s
}

// SumNumbers sums the values of map m. Its supports both integers
// and floats as map values.
func SumNumbers[K comparable, V Number](m map[K]V) V {
    var s V
    for _, v := range m {
        s += v
    }
    return s
}
```
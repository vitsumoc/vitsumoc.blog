---
title: "[未完成]一个Go中的闭包问题"
url: 一个Go中的闭包问题
date: 2024-09-03 16:19:35
categories:
- go
tags:
- go
---

```go
// 用来记录找到节点的数据
var resultKt *model.KnowTree
resultList := make([]*model.KnowTree, 0)
resultMap := make(map[int]*model.KnowTree)
fmt.Println(id)

// 先找到根节点
for _, kt := range kts {
	if *kt.Id == id {
		fmt.Println(*kt.Id)
		resultKt = &kt
	}
}
fmt.Println(resultKt.ToDebug())
```
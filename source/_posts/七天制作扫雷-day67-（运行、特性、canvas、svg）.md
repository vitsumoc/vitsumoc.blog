---
title: 七天制作扫雷-day67+（运行、特性、canvas、svg）
url: 七天扫雷day6789
date: 2024-06-04 14:26:49
categories:
- 七天扫雷
tags:
- 前端
- 小玩具
---

与文档配套的源码和资源在[这里](https://github.com/vitsumoc/7daysMineSweeper)

<!-- more -->

# day6 游戏运行

## 预加载图片资源

在 `settings.js` 中，预加载游戏里会使用的图片资源，可以消除初次获取图片时的闪烁感。

```js
// 预加载图片资源 (用于生成缓存)
var img1 = new Image();
img1.src = "asset/" + CS_IDLE + ".png";
var img2 = new Image();
img2.src = "asset/" + CS_FLAG + ".png";
var img3 = new Image();
img3.src = "asset/" + CS_BOMB + ".png";
var img4 = new Image();
img4.src = "asset/" + CS_0 + ".png";
var img5 = new Image();
img5.src = "asset/" + CS_1 + ".png";
var img6 = new Image();
img6.src = "asset/" + CS_2 + ".png";
var img7 = new Image();
img7.src = "asset/" + CS_3 + ".png";
var img8 = new Image();
img8.src = "asset/" + CS_4 + ".png";
var img9 = new Image();
img9.src = "asset/" + CS_5 + ".png";
var img10 = new Image();
img10.src = "asset/" + CS_6 + ".png";
var img11 = new Image();
img11.src = "asset/" + CS_7 + ".png";
var img12 = new Image();
img12.src = "asset/" + CS_8 + ".png";
```

> 可以通过Chrome的网络调试工具，确认我们在进入页面时已经加载了所有想要的图片资源。

## 加载第三方UI

因为 `app` 类只需求了信息显示的回调函数，没有实际的实现信息显示功能，在运行游戏时我们需要提供一个实际的信息显示功能。

这里选用了三方库 `layer` 来实现消息弹窗，`layer` 依赖 `jquery`，因此同时引入了 `jquery`。

```js
<script src="jquery/jquery-3.6.0.min.js"></script>
<script src="layer/layer.js"></script>
<script src="settings.js"></script>
<script src="board.js"></script>
<script src="cell.js"></script>
<script src="view_dom.js"></script>
<script src="game.js"></script>
<script>
    var container = document.getElementById("container");
    app.init(container, 10, 10, 20, layer.msg);
</script>
```

到此处，游戏已经可以运行了。

# day7 加入特性

## 难度选择菜单、开始按钮

扩展 `app` 类的接口，并在页面上放上相应的按钮就可以实现。

## 高分排行榜

通过 `localStorage` 实现。

## 黄色圆脸表情

在页面上添加一个 `dom` 元素用来盛放，同时给 `app` 对象加一个输赢状态的回调。

# day8 实现基于 canvas 的显示类

可以用于辅助理解 `canvas` 工作原理，请直接参考 `canvas` 相关文档和 `view_canvas.js`。

# day9 实现基于 svg 的显示类

可以用于辅助理解 `svg` 工作原理，请直接参考 `svg` 相关文档和 `view_dom.js`。
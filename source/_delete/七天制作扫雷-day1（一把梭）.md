---
title: 七天制作扫雷-day1（一把梭）
url: 七天扫雷day1
date: 2024-06-04 10:22:31
categories:
- 七天扫雷
tags:
- 前端
- 小玩具
---

> 这一系列文档是曾经写给一个想要学编程的朋友，他当时在自学前端，学习了HTML、CSS和JavaScript，但是并不知道这些东西有什么作用，于是我就写了这些文档给他，可惜的是，最后他还是没有成为一名程序员。

与文档配套的源码和资源在[这里](https://github.com/vitsumoc/7daysMineSweeper)

<!-- more -->

# 简介

面向过程的编程方式比较直观，易理解。每个开发步骤中需要解决的问题和编写的代码一一对应，一气呵成的代码编写会带来一种畅爽的感觉。

按照正向推进的思路，要制作一个扫雷游戏我们需要：

1. 创建棋盘
2. 划分格子
3. 埋雷
4. 点击格子事件
5. 格子打开事件

接下来我们就来一口气实现一个最简单的扫雷游戏。

# 实现

## 1. 绘制棋盘

基础的html页面，设置了名为container的div，作为扫雷游戏的棋盘。

用css给背景和棋盘设置不同的颜色，用flex布局将棋盘居中摆放。

### 代码

```html
<!DOCTYPE html>
<html>
	<head>
		<meta charset="utf-8" />
		<meta name="viewport" content="width=device-width, initial-scale=1">
		<title>扫雷</title>
		<style type="text/css">
			* {
				margin: 0;
				padding: 0;
				user-select: none;
			}
			body {
				width: 100vw;
				height: 100vh;
				display: flex;
				justify-content: center;
				align-items: center;
				box-sizing: border-box;
				background-color: #333;
			}
			#container {
				height: 600px;
				width: 600px;
				background-color: #f7f7f7;
			}
		</style>
	</head>
	<body>
		<div id="container"></div>
	</body>
</html>
```

### 效果

![](1.png)

## 2. 划格子

本例中，用dom元素制作扫雷游戏中的格子，循环创建足够数量的div，并放入dom树。

用css代码绘制格子的边线，在div中插入*字符表示这是一个未打开的格子。

### 代码

```html
<!DOCTYPE html>
<html>
	<head>
		<meta charset="utf-8" />
		<meta name="viewport" content="width=device-width, initial-scale=1">
		<title>扫雷</title>
		<style type="text/css">
			* {
				margin: 0;
				padding: 0;
				user-select: none;
			}
			body {
				width: 100vw;
				height: 100vh;
				display: flex;
				justify-content: center;
				align-items: center;
				box-sizing: border-box;
				background-color: #333;
			}
			#container {
				height: 600px;
				width: 600px;
				background-color: #f7f7f7;
				display: flex;
				flex-wrap: wrap;
			}
			.item {
				width: 30px;
				height: 30px;
				border: 1px solid #332200;
				box-sizing: border-box;
				display: flex;
				justify-content: center;
				align-items: center;
			}
		</style>
	</head>
	<body>
		<div id="container"></div>
	</body>
	<script type="text/javascript">
		// 行
		var row_count = 20;
    // 列
    var column_count = 20;
    // 棋盘
		var container = document.getElementById('container')
    // 所有格子的容器，二维数组
		var doms = []
    // 循环每一行
		for (var x = 0; x < row_count; x++) {
     	// 内层数组，表示一行
			var row = []
      // 循环每一列
			for (var y = 0; y < column_count; y++) {
        // 创建一个新的div dom元素
				var div = document.createElement('div')
        // 设置class 获得css样式
				div.setAttribute('class', 'item');
        // 创建格子对象，放入行数组中
        // 格子对象包括的内容有：div元素的引用、是否是雷的标识、是否已经打开的标识
				row.push({dom: div, is_boom: false, opened: false})
        // 将div元素放入dom树中，才能在页面中呈现
				container.appendChild(div)
			}
      // 将行放入容器中
			doms.push(row)
		}
		
		// 初始化每一个格子的显示
    // 循环格子容器
		for (var x in doms) {
			// 获得行
      var row = doms[x]
			// 循环行
      for (var y in row) {
				// 对每一个格子对象进行操作，将html文本设置为*
        row[y].dom.innerHTML = '*'
			}
		}
	</script>
</html>
```

### 效果

![](2.png)

## 3. 埋雷

需要随机的在格子里生成若干颗雷，定义变量boom_count表示雷的数量。

通过一个while循环，重复遍历所有的格子，每次处理都有2%的概率给当前格子放入地雷。当埋雷的数量等于设置的雷总数时，结束循环。

调试时，将有雷的格子字符修改为+，方便查看创建雷的效果。

### 代码

```html
<!DOCTYPE html>
<html>
	<head>
		<meta charset="utf-8" />
		<meta name="viewport" content="width=device-width, initial-scale=1">
		<title>扫雷</title>
		<style type="text/css">
			* {
				margin: 0;
				padding: 0;
				user-select: none;
			}
			body {
				width: 100vw;
				height: 100vh;
				display: flex;
				justify-content: center;
				align-items: center;
				box-sizing: border-box;
				background-color: #333;
			}
			#container {
				height: 600px;
				width: 600px;
				background-color: #f7f7f7;
				display: flex;
				flex-wrap: wrap;
			}
			.item {
				width: 30px;
				height: 30px;
				border: 1px solid #332200;
				box-sizing: border-box;
				display: flex;
				justify-content: center;
				align-items: center;
			}
		</style>
	</head>
	<body>
		<div id="container"></div>
	</body>
	<script type="text/javascript">
		// 行
		var row_count = 20;
    // 列
    var column_count = 20;
    // 设置雷数 50
    var boom_count = 50;
		var container = document.getElementById('container')
		var doms = []
		for (var x = 0; x < row_count; x++) {
			var row = []
			for (var y = 0; y < column_count; y++) {
				var div = document.createElement('div')
				div.setAttribute('class', 'item');
				row.push({dom: div, is_boom: false, opened: false})
				container.appendChild(div)
			}
			doms.push(row)
		}
		
		//初始化覆盖每一个雷点
		for (var x in doms) {
			var row = doms[x]
			for (var y in row) {
				row[y].dom.innerHTML = '*'
			}
		}
		
		// 初始化每一颗雷
    // 已经创建的雷数量
		var inited_boom = 0
		// 当前循环序号
    var current_time = 0
    // 当创建的雷数小于设置的雷数
		while (inited_boom < boom_count) {
			//通过序号获取当前点
			var index = current_time % (row_count * column_count)
			var row = parseInt(index / column_count)
			var column = index % column_count
			// 格子对象
			var item = doms[row][column]
			//随机生成雷 2%概率
			if (Math.random() < 0.02) {
				if (!item.is_boom) {
					item.is_boom = true
					inited_boom++
					item.dom.innerHTML = '+'
				}
			}
			//进入下一个点
			current_time++
		}
	</script>
</html>
```

### 效果

每次刷新都会看到随机分布的50个加号，表示我们埋雷的位置。

![](3.png)

## 4. 点击格子

面向过程的编程思路就是这样的简单粗暴，只要完成点击格子的逻辑，扫雷就完成了。

这里对格子中的雷进行了判断，对已经打开的格子进行了判断，并声明了新的全局变量fail，记录玩家是否已经失败。

对于没有雷的格子，使用open_this方法打开，这个方法比较复杂，这种我们可以先标记为// TODO。

### 代码

```html
<!DOCTYPE html>
<html>
	<head>
		<meta charset="utf-8" />
		<meta name="viewport" content="width=device-width, initial-scale=1">
		<title>扫雷</title>
		<style type="text/css">
			* {
				margin: 0;
				padding: 0;
				user-select: none;
			}
			body {
				width: 100vw;
				height: 100vh;
				display: flex;
				justify-content: center;
				align-items: center;
				box-sizing: border-box;
				background-color: #333;
			}
			#container {
				height: 600px;
				width: 600px;
				background-color: #f7f7f7;
				display: flex;
				flex-wrap: wrap;
			}
			.item {
				width: 30px;
				height: 30px;
				border: 1px solid #332200;
				box-sizing: border-box;
				display: flex;
				justify-content: center;
				align-items: center;
			}
		</style>
	</head>
	<body>
		<div id="container"></div>
	</body>
	<script type="text/javascript">
		// 行
		var row_count = 20;
    // 列
    var column_count = 20;
    // 雷
    var boom_count = 50;
    // 失败
    var fail = false;
		var container = document.getElementById('container')
		var doms = []
		for (var x = 0; x < row_count; x++) {
			var row = []
			for (var y = 0; y < column_count; y++) {
				var div = document.createElement('div')
				div.setAttribute('class', 'item');
				row.push({dom: div, is_boom: false, opened: false})
				container.appendChild(div)
			}
			doms.push(row)
		}
		
		//初始化覆盖每一个雷点
		for (var x in doms) {
			var row = doms[x]
			for (var y in row) {
				row[y].dom.innerHTML = '*'
			}
		}
		
		//初始化每一颗雷
		var inited_boom = 0
		var current_time = 0
		while (inited_boom < boom_count) {
			//通过序号获取当前点
			var index = current_time % (row_count * column_count)
			var row = parseInt(index / column_count)
			var column = index % column_count
			
			var item = doms[row][column]
			//随机生成雷 2%概率
			if (Math.random() < 0.02) {
				if (!item.is_boom) {
					item.is_boom = true
					inited_boom++
					//item.dom.innerHTML = '+'
				}
			}
			//进入下一个点
			current_time++
		}
		
		//当方块被点击
		for (var x in doms) {
			(function(x) {
				var row = doms[x]
				for (var y in row) {
					(function(y) {
						var item = row[y]
						item.dom.addEventListener('click', function() {
							//已经输了
							if (fail) {
								alert('你输了，刷新吧')
								return
							}
							//点雷结束
							if (item.is_boom) {
								fail = true
								alert('你输了，刷新吧')
								return
							}
							//已经打开过了
							if (item.opened) {
								return
							}
							//如果不是雷，就触发打开动作，递归的显示一片区域
							open_this(x, y)
						})
					} (y))
				}
			} (x))
		}
		
		function open_this(x, y) {
			// TODO
      // 递归的打开格子，并且判断是否胜利
		}
	</script>
</html>
```

### 效果

点中有雷的格子会导致游戏失败，并且无法再点击其他格子。

因为 open_this 方法还未编写，此时点中没有雷的格子时无事发生。

![](4.png)

## 5. 打开格子

扫雷游戏的逻辑是，当打开一个格子时：

1. 如果格子是雷，则游戏失败
2. 如果格子周围有雷，则显示周围的雷数量
3. 如果格子周围没有雷，则打开此格子，并且按照上述的逻辑同时处理周围的八个格子
4. 如果所有非雷的格子都被打开，则游戏胜利

在逻辑3中，出现了不确定层数的循环处理，此处可以通过递归的方式解决。

### 完整代码

```html
<!DOCTYPE html>
<html>
	<head>
		<meta charset="utf-8" />
		<meta name="viewport" content="width=device-width, initial-scale=1">
		<title>扫雷</title>
		<style type="text/css">
			* {
				margin: 0;
				padding: 0;
				user-select: none;
			}
			body {
				width: 100vw;
				height: 100vh;
				display: flex;
				justify-content: center;
				align-items: center;
				box-sizing: border-box;
				background-color: #333;
			}
			#container {
				height: 600px;
				width: 600px;
				background-color: #f7f7f7;
				display: flex;
				flex-wrap: wrap;
			}
			.item {
				width: 30px;
				height: 30px;
				border: 1px solid #332200;
				box-sizing: border-box;
				display: flex;
				justify-content: center;
				align-items: center;
			}
		</style>
	</head>
	<body>
		<div id="container"></div>
	</body>
	<script type="text/javascript">
		// 行
		var row_count = 20;
    // 列
    var column_count = 20;
    // 雷
    var boom_count = 50;
    // 失败
    var fail = false;
    // 打开区域
    var clear_area = 0;
		var container = document.getElementById('container')
		var doms = []
		for (var x = 0; x < row_count; x++) {
			var row = []
			for (var y = 0; y < column_count; y++) {
				var div = document.createElement('div')
				div.setAttribute('class', 'item');
				row.push({dom: div, is_boom: false, opened: false})
				container.appendChild(div)
			}
			doms.push(row)
		}
		
		//初始化覆盖每一个雷点
		for (var x in doms) {
			var row = doms[x]
			for (var y in row) {
				row[y].dom.innerHTML = '*'
			}
		}
		
		//初始化每一颗雷
		var inited_boom = 0
		var current_time = 0
		while (inited_boom < boom_count) {
			//通过序号获取当前点
			var index = current_time % (row_count * column_count)
			var row = parseInt(index / column_count)
			var column = index % column_count
			
			var item = doms[row][column]
			//随机生成雷 2%概率
			if (Math.random() < 0.02) {
				if (!item.is_boom) {
					item.is_boom = true
					inited_boom++
					//item.dom.innerHTML = '+'
				}
			}
			//进入下一个点
			current_time++
		}
		
		//当方块被点击
		for (var x in doms) {
			(function(x) {
				var row = doms[x]
				for (var y in row) {
					(function(y) {
						var item = row[y]
						item.dom.addEventListener('click', function() {
							//已经输了
							if (fail) {
								alert('你输了，刷新吧')
								return
							}
							//点雷结束
							if (item.is_boom) {
								fail = true
								alert('你输了，刷新吧')
								return
							}
							//已经打开过了
							if (item.opened) {
								return
							}
							//如果不是雷，就触发打开动作，递归的显示一片区域
							open_this(x, y)
						})
					} (y))
				}
			} (x))
		}
		
		function open_this(x, y) {
			var x = Number(x)
			var y = Number(y)
			//已经打开，无事发生
			if (doms[x][y].opened) {
				return
			}
			//标记打开
			doms[x][y].opened = true
			//计算身边雷的数量，并且填写数字的方法
			if (show_boom_count(x, y) == 0) {
				//遍历身边的方块，只要不是雷就全部打开
				for (var xx = x - 1; xx < x + 2; xx++) {
					for (var yy = y - 1; yy < y + 2; yy++) {
						if (doms[xx] && doms[xx][yy]) {
							if (!doms[xx][yy].is_boom && !doms[xx][yy].opened) {
								open_this(xx, yy)
							}
						}
					}
				}
			}
			//记录已经打开的方块数量，全部打开则胜利
			clear_area++
			if (clear_area + boom_count == (row_count * column_count)) {
				alert('你赢了，大神，刷新吧')
			}
		}
		
		function show_boom_count(x, y) {
			var boom_count = 0
			for (var xx = x - 1; xx < x + 2; xx++) {
				for (var yy = y - 1; yy < y + 2; yy++) {
					if (doms[xx] && doms[xx][yy]) {
						if (doms[xx][yy].is_boom) {
							boom_count++
						}
					}
				}
			}
			if (boom_count > 0) {
				doms[x][y].dom.innerHTML = boom_count
			} else {
				doms[x][y].dom.innerHTML = ''
			}
			//根据雷的多少赋予颜色
			if (boom_count < 3) {
				doms[x][y].dom.style.color = "#33DD00"
			} else if (boom_count < 6) {
				doms[x][y].dom.style.color = "#DDDD00"
			} else {
				doms[x][y].dom.style.color = "#DD3300"
			}
			return boom_count
		}
	</script>
</html>
```

# 总结

正向推进代码就是这样，简单、粗暴、畅爽。

这个扫雷游戏的demo十分简单(算上注释不到200行)，但是已经难以修改维护了。例如，dom元素也许不是很适合做游戏，当前的图像显示效果也过于简陋，如果想在这个代码的基础上改为SVG显示或Canvas显示，估计要费一些的功夫。

接下来的步骤，我们会尽量让代码更加模块化，虽然代码量会增加，但是模块化之后才方便我们进行各种拓展。
---
title: Matplotlib入门笔记
url: pythonMatplotlib
date: 2023-11-29 15:40:57
categories:
- python
tags:
- python
- 库
- 笔记
- 豆知识
---

# 前言

网站 https://matplotlib.org/stable/

Matplotlib 是一个用于创建静态、动画和交互式可视化的综合库。

本文是学习 Matplotlib 过程中的笔记，所有内容都来自官方文档：https://matplotlib.org/stable/users/explain/quick_start.html

<!-- more -->

# 1. 入门示例

```python python
import matplotlib.pyplot as plt
import numpy as np

import matplotlib as mpl

def e1():
  x = np.linspace(0, 2 * np.pi, 200)
  y = np.sin(x)

  fig, ax = plt.subplots()
  ax.plot(x, y)
  plt.show()

e1()
```

# 2. 窗口、图像和绘制

```python python
def e2():
  # 创建一个只有一个 axes 的 figure
  fig, ax = plt.subplots()
  # 在 axes 上 plot 一些数据
  ax.plot([1, 2, 3, 4], [1, 4, 2, 3])
  plt.show()

e2()
```

# 3. figure 的构成部分

```python python
# figure 是一个绘图窗口
# axes 是一副数据图像
# axis 是坐标轴
def e3():
  # 一个没有 axes 的 figure
  fig = plt.figure()
  # 只有一个 axes 的图像
  fig, ax = plt.subplots()
  # 2 * 2 布局的图像
  fig, axs = plt.subplots(2, 2)
  # 左一右二布局
  fig, axs = plt.subplot_mosaic([['left', 'right_top'], ['left', 'right_bottom']])
  plt.show()

e3()
```

# 4. 输入数据类型

```python python
def e4():
  # plot 接受 np.array np.ma.masked_array np.asarray 三种类型的输入
  # 如果不是此类数据，需要先进行处理
  b = np.matrix([[1, 2], [3, 4]])
  b_asarray = np.asarray(b)
  # 对于一些已经准备好的对象(字典)数据, 也可以用下面的方式输入
  np.random.seed(19680801)  # seed the random number generator.
  # a 是 0-50 的整数 用于每个数据的 x 坐标
  # b 是 50个随机数 用于每个数据的 y 坐标 (50个0-1的随机数 * 10 再加 x坐标)
  # c 随机颜色 50个50以下的整数
  # d 是随机尺寸
  data = {'a': np.arange(50),
          'c': np.random.randint(0, 50, 50),
          'd': np.random.randn(50)}
  data['b'] = data['a'] + 10 * np.random.randn(50)
  data['d'] = np.abs(data['d']) * 100

  fig, ax = plt.subplots(figsize=(5, 2.7), layout='constrained')
  # 离散数据 使用字典中的内容赋值
  ax.scatter('a', 'b', c='c', s='d', data=data)
  ax.set_xlabel('entry a')
  ax.set_ylabel('entry b')
  plt.show()

e4()
```

# 5. 接口风格

```python python
# mplib提供了两种接口风格 一是显示的获取各层对象并调用 二是直接使用plt搞定一切
# 显示风格的例子
def e5_1():
  x = np.linspace(0, 2, 100)  # 示意数据
  # 获得 figure 和 axes
  fig, ax = plt.subplots(figsize=(5, 2.7), layout='constrained')
  # 一次二次和三次函数
  ax.plot(x, x, label='linear')
  ax.plot(x, x**2, label='quadratic')
  ax.plot(x, x**3, label='cubic')
  # 指定xy的label
  ax.set_xlabel('x label')
  ax.set_ylabel('y label')
  # axes 的title
  ax.set_title("Simple Plot")
  # 添加一个图例 用来显示各plot的label
  ax.legend()
  plt.show()

# 隐式风格的例子 效果和显示风格完全相同
def e5_2():
  x = np.linspace(0, 2, 100)
  plt.figure(figsize=(5, 2.7), layout='constrained')
  plt.plot(x, x, label='linear')
  plt.plot(x, x**2, label='quadratic')
  plt.plot(x, x**3, label='cubic')
  plt.xlabel('x label')
  plt.ylabel('y label')
  plt.title("Simple Plot")
  plt.legend()
  plt.show()

e5_1()
e5_2()
```

# 6. 制作辅助函数

```python python
# 制作工具函数, 避免代码重复
def e6_plotter(ax, data1, data2, param_dict):
    """
    A helper function to make a graph.
    """
    out = ax.plot(data1, data2, **param_dict)
    return out

def e6():
  data1, data2, data3, data4 = np.random.randn(4, 100)  # make 4 random data sets
  fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(5, 2.7))
  e6_plotter(ax1, data1, data2, {'marker': 'x'})
  e6_plotter(ax2, data3, data4, {'marker': 'o'})
  plt.show()

e6()
```

# 7. 样式

```python python
def e7():
  data1, data2 = np.random.randn(2, 100)
  fig, ax = plt.subplots(figsize=(5, 2.7))
  x = np.arange(len(data1))
  # plot 直接跟样式参数
  ax.plot(x, np.cumsum(data1), color='blue', linewidth=3, linestyle='--')
  l, = ax.plot(x, np.cumsum(data2), color='orange', linewidth=2)
  # plot 后对返回内容进行样式赋值
  l.set_linestyle(':')
  plt.show()

e7()
```

# 8. 标记

## 基础标记

```python python
def e8_1():
  mu, sigma = 115, 15
  # x 是一万个值的列表 randn 会给出一组正态分布的随机数结果
  x = mu + sigma * np.random.randn(10000)
  fig, ax = plt.subplots(figsize=(5, 2.7), layout='constrained')
  # the histogram of the data
  # 直方图参数: x-数据内容 50-柱数量 density-返回概率密度 
  n, bins, patches = ax.hist(x, 50, density=True, facecolor='C0', alpha=0.75)

  # 轴和图的标题
  ax.set_xlabel('Length [cm]')
  ax.set_ylabel('Probability')
  ax.set_title('Aardvark lengths\n (not really)')
  # 文本 (使用了数学符号)
  ax.text(75, .025, r'$\mu=115,\ \sigma=15$')
  # 轴定义
  ax.axis([55, 175, 0, 0.03])
  # 网线
  ax.grid(True)
  plt.show()

e8_1()
```

## 标记图上的点

```python python
def e8_2():
  fig, ax = plt.subplots(figsize=(5, 2.7))

  t = np.arange(0.0, 5.0, 0.01)
  s = np.cos(2 * np.pi * t)
  line, = ax.plot(t, s, lw=2)
  # 使用点位、文本位、箭头设置来标记点
  ax.annotate('local max', xy=(2, 1), xytext=(3, 1.5),
            arrowprops=dict(facecolor='black', shrink=0.05))
  # y轴limit
  ax.set_ylim(-2, 2)
  plt.show()

e8_2()
```

## 添加 Legend用以区分数据

```python python
def e8_3():
  data1, data2, data3 = np.random.randn(3, 100)
  fig, ax = plt.subplots(figsize=(5, 2.7))
  ax.plot(np.arange(len(data1)), data1, label='data1')
  ax.plot(np.arange(len(data2)), data2, label='data2')
  ax.plot(np.arange(len(data3)), data3, 'd', label='data3')
  ax.legend()
  plt.show()

e8_3()
```

# 9. 轴的比例和刻度

## 轴的比例定义

```python python
def e9_1():
  # 100 个随机数
  data1 = np.random.randn(100)
  # 两个 axes
  fig, axs = plt.subplots(1, 2, figsize=(5, 2.7), layout='constrained')
  # x轴为随机数的数量
  xdata = np.arange(len(data1))
  # y数据为 10 ** data1
  data = 10**data1
  # axes 使用折线图
  axs[0].plot(xdata, data)
  # axes 使用对数坐标 图像内容接近 data1 的原始值
  axs[1].set_yscale('log')
  axs[1].plot(xdata, data)
  plt.show()

e9_1()
```

## 手动操作 axis 上的 ticks

```python python
def e9_2():
  data1 = np.random.randn(100)
  xdata = np.arange(len(data1))
  fig, axs = plt.subplots(2, 1, layout='constrained')
  axs[0].plot(xdata, data1)
  axs[0].set_title('Automatic ticks')

  axs[1].plot(xdata, data1)
  # 设置x 轴和显示内容
  axs[1].set_xticks(np.arange(0, 100, 30), ['zero', '30', 'sixty', '90'])
  # 设置 y 轴
  axs[1].set_yticks([-1.5, 0, 1.5])
  axs[1].set_title('Manual ticks')
  plt.show()

e9_2()
```

## 使用时间做轴

```python python
def e9_3():
  fig, ax = plt.subplots(figsize=(5, 2.7), layout='constrained')
  # 通过时间范围和间隔构建时间戳数组
  dates = np.arange(np.datetime64('2021-11-15'), np.datetime64('2021-12-25'),
                    np.timedelta64(1, 'h'))
  # 随机数的数据
  data = np.cumsum(np.random.randn(len(dates)))
  # x 和 y 数据正常放入图像
  ax.plot(dates, data)
  # 设置日期格式化方式并添加到轴
  cdf = mpl.dates.ConciseDateFormatter(ax.xaxis.get_major_locator())
  ax.xaxis.set_major_formatter(cdf)
  plt.show()

e9_3()
```

## 使用字符串做轴

```python python
def e9_4():
  fig, ax = plt.subplots(figsize=(5, 2.7), layout='constrained')
  categories = ['turnips', 'rutabaga', 'cucumber', 'pumpkins']

  ax.bar(categories, np.random.rand(len(categories)))
  plt.show()

e9_4()
```

## 添加更多的轴

```python python
def e9_5():
  t = np.arange(0.0, 5.0, 0.01)
  s = np.cos(2 * np.pi * t)
  # fig上的图像为 ax1 和 ax3
  fig, (ax1, ax3) = plt.subplots(1, 2, figsize=(7, 2.7), layout='constrained')
  l1, = ax1.plot(t, s)
  # ax2 和 ax1 绘制在一起, 共享x轴
  ax2 = ax1.twinx()
  l2, = ax2.plot(t, range(len(t)), 'C1')
  ax2.legend([l1, l2], ['Sine (left)', 'Straight (right)'])

  ax3.plot(t, s)
  ax3.set_xlabel('Angle [rad]')
  # secondary_xaxis 用于创建一个新的x轴 传入了和原x轴的互相转换函数
  ax4 = ax3.secondary_xaxis('top', functions=(np.rad2deg, np.deg2rad))
  ax4.set_xlabel('Angle [°]')
  plt.show()

e9_5()
```

# 10. 色块图

```python python
def e10():
  data1, data2, data3 = np.random.randn(3, 100)
  X, Y = np.meshgrid(np.linspace(-3, 3, 128), np.linspace(-3, 3, 128))
  Z = (1 - X/2 + X**5 + Y**3) * np.exp(-X**2 - Y**2)

  fig, axs = plt.subplots(2, 2, layout='constrained')
  pc = axs[0, 0].pcolormesh(X, Y, Z, vmin=-1, vmax=1, cmap='RdBu_r')
  fig.colorbar(pc, ax=axs[0, 0])
  axs[0, 0].set_title('pcolormesh()')

  co = axs[0, 1].contourf(X, Y, Z, levels=np.linspace(-1.25, 1.25, 11))
  fig.colorbar(co, ax=axs[0, 1])
  axs[0, 1].set_title('contourf()')

  pc = axs[1, 0].imshow(Z**2 * 100, cmap='plasma',
                            norm=mpl.colors.LogNorm(vmin=0.01, vmax=100))
  fig.colorbar(pc, ax=axs[1, 0], extend='both')
  axs[1, 0].set_title('imshow() with LogNorm()')

  pc = axs[1, 1].scatter(data1, data2, c=data3, cmap='RdBu_r')
  fig.colorbar(pc, ax=axs[1, 1], extend='both')
  axs[1, 1].set_title('scatter()')
  plt.show()

e10()
```

# 11. 多 axes 使用 dict 操作

```python python
def e11():
  fig, axd = plt.subplot_mosaic([['upleft', 'right'],
                               ['lowleft', 'right']], layout='constrained')
  axd['upleft'].set_title('upleft')
  axd['lowleft'].set_title('lowleft')
  axd['right'].set_title('right')

e11()
```
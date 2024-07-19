---
title: Numpy入门笔记
url: pythonNumpy
date: 2023-11-28 09:13:53
categories:
- python
tags:
- python
- 库
- 笔记
- 豆知识
---

# 前言

网站 https://numpy.org/

NumPy（Numerical Python）是Python中数字处理的事实标准，也是学习其他数据知识的必备工具。

本文是学习Numpy过程中的笔记，所有内容都来自官方文档：https://numpy.org/doc/stable/user/absolute_beginners.html

<!-- more -->

# 1. 普通数组和np数组的区别

```python python
import numpy as np

# 普通数组
a = [0, 1, 2, 3]
print(a)
# np数组
b = np.array(a)
print(b)
```

# 2. 创建np数组的方法

```python python
np.zeros(2) # 全0填充
np.ones(2) # 全1填充
np.empty(2) # 空数组
np.arange(4) # [0, 1, 2, 3]
np.arange(2, 9, 2) # [2, 4, 6, 8]
np.linspace(0, 10, num=5) # [0, 2.5, 5, 7.5, 10]
# 可以自己决定数据类型
np.ones(2, dtype=np.int64) # [1, 1]
```

# 3. 排序和拼接

```python python
arr = np.array([2, 1, 5, 3, 7, 4, 6, 8])
np.sort(arr) # 排序
a = np.array([1, 2, 3, 4])
b = np.array([5, 6, 7, 8])
np.concatenate((a, b)) # 拼接
x = np.array([[1, 2], [3, 4]])
y = np.array([[5, 6]])
np.concatenate((x, y), axis=0) # 拼接
```

# 4. 形状和大小

```python python
array_example = np.array([
  [[0, 1, 2, 3], [4, 5, 6, 7]],
  [[0, 1, 2, 3], [4, 5, 6, 7]],
  [[0 ,1 ,2, 3], [4, 5, 6, 7]]
])
array_example.ndim # 维度 3
array_example.size # 大小 24
array_example.shape # 形状 (3, 2, 4)
```

# 5. 改变数组的形状

```python python
a = np.arange(6)
b = a.reshape(3, 2)
# [[0 1]
#  [2 3]
#  [4 5]]
np.reshape(a, newshape=(1, 6), order='C') # 更多参数
```

# 6. 添加维度

```python python
a = np.array([1, 2, 3, 4, 5, 6])
a.shape # 一维 (6, )
a2 = a[np.newaxis, :]
a2.shape # 二维 (1, 6)
col_vector = a[:, np.newaxis] # 插入列向量
col_vector.shape # 二维 (6, 1)
# 在指定维度插入
b = np.expand_dims(a, axis=1)
b.shape # (6, 1)
c = np.expand_dims(a, axis=0)
c.shape # (1, 6)
```

# 7. 索引和切片

```python python
data = np.array([1, 2, 3])
data[1] # 正常索引方式 2
data[0:2] # 正常切片 array([1, 2])
data[1:] # 正向到底 array([2, 3])
data[-2:] # 反向到底 array([2, 3])
# 条件过滤
a = np.array([[1, 2, 3, 4], [5, 6, 7, 8], [9, 10, 11, 12]])
a[a < 8] # 符合条件的内容 [1 2 3 4 5 6 7]
five_up = (a >= 5) # 条件表达式作为参数
a[five_up] # [5 6 7 8 9 10 11 12]
c = a[(a > 2) & (a < 11)] # 可以使用与&或| [3 4 5 6 7 8 9 10]
five_up = (a > 5) | (a == 5) # 条件本身会被计算成一个bool数组, 和原数组结构相同
five_up
# [[False False False False]
#  [ True  True  True  True]
#  [ True  True  True True]]
a = np.array([[1, 2, 3, 4], [5, 6, 7, 8], [9, 10, 11, 12]])
b = np.nonzero(a < 6) # 根据查询条件, 返回符合条件的元素的索引
# 返回的数组数是a的维数, 返回值是索引值, 返回长度是符合条件的个数
# print(b) # (array([0, 0, 0, 0, 1], dtype=int64), array([0, 1, 2, 3, 0], dtype=int64))
# 将上述内容压缩成坐标列表
list_of_coordinates= list(zip(b[0], b[1])) # [[0, 0], [0, 1], [0, 2], [0, 3], [1, 0]]
a[b] # 也可以用索引直接获得元素 [1 2 3 4 5]
# 结果为空
not_there = np.nonzero(a == 42) # (array([], dtype=int64), array([], dtype=int64))
```

# 8. 现有数据转数组

```python python
a = np.array([1,  2,  3,  4,  5,  6,  7,  8,  9, 10])
arr1 = a[3:8] # 通过切片创建新数组 array([4, 5, 6, 7, 8])
a1 = np.array([[1, 1], [2, 2]])
a2 = np.array([[3, 3], [4, 4]])
np.vstack((a1, a2)) # 垂直堆叠 [[1, 1], [2, 2], [3, 3], [4, 4]]
np.hstack((a1, a2)) # 水平堆叠 [[1, 1], [3, 3], [2, 2], [4, 4]]
x = np.arange(1, 25).reshape(2, 12) # 素材
# array([[ 1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11, 12],
#       [13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24]])
np.hsplit(x, 3) # 拆成三个
# [array([[ 1,  2,  3,  4],
#        [13, 14, 15, 16]]), array([[ 5,  6,  7,  8],
#        [17, 18, 19, 20]]), array([[ 9, 10, 11, 12],
#        [21, 22, 23, 24]])]
np.hsplit(x, (3, 4)) # 按指定列号拆分
# [array([[ 1,  2,  3],
#        [13, 14, 15]]), array([[ 4],
#        [16]]), array([[ 5,  6,  7,  8,  9, 10, 11, 12],
#        [17, 18, 19, 20, 21, 22, 23, 24]])]
# 视图是引用, 修改视图也会修改原数据
a = np.array([[1, 2, 3, 4], [5, 6, 7, 8], [9, 10, 11, 12]])
b1 = a[0, :] # array([1, 2, 3, 4])
b1[0] = 99
a
# array([[99,  2,  3,  4],
#        [ 5,  6,  7,  8],
#        [ 9, 10, 11, 12]])
# copy是复制, 修改copy对原数据没影响
b2 = a.copy()
```

# 9. 基础数组操作

```python python
# 加减乘除
data = np.array([1, 2]) # [1 2]
ones = np.ones(2, dtype=int) # [1 1]
data + ones # [2 3]
# 求和
a = np.array([1, 2, 3, 4])
a.sum() # 10
# 在所选维度求和
b = np.array([[1, 1], [2, 2]])
b.sum(axis=0) # [3, 3]
b.sum(axis=1) # [2, 4]
# 和常量的运算
data = np.array([1.0, 2.0])
data * 1.6 # [1.6 3.2]
# 素材
a = np.array([[0.45053314, 0.17296777, 0.34376245, 0.5510652],
              [0.54627315, 0.05093587, 0.40067661, 0.55645993],
              [0.12697628, 0.82485143, 0.26590556, 0.56917101]])
a.sum() # 求和 4.8595784
a.min() # 极小值 0.05093587
a.min(axis=0) # 维度极小值 [0.12697628, 0.05093587, 0.26590556, 0.5510652 ]
```

# 10. 矩阵

```python python
data = np.array([[1, 2], [3, 4], [5, 6]])
# array([[1, 2],
#        [3, 4],
#        [5, 6]])
data[0, 1] # 正常索引 2
data[1:3] # 正常切片 array([[3, 4], [5, 6]])
data[0:2, 0] # 0:2是切片, 0是索引, 切片和索引混用 array([1, 3])
data.max() # 6
data.min() # 1
data.sum() # 21
# 也可以指定维度
data = np.array([[1, 2], [5, 3], [4, 6]])
data.max(axis=0) # array([5, 6])
data.max(axis=1) # array([2, 5, 6])
# 矩阵之间的运算（需要矩阵尺寸相同）
data = np.array([[1, 2], [3, 4]])
ones = np.array([[1, 1], [1, 1]])
data + ones # array([[2, 3], [4, 5]])
# 如果某个矩阵只有一行或者一列, 也可使用广播规则运算
data = np.array([[1, 2], [3, 4], [5, 6]])
ones_row = np.array([[1, 1]])
data + ones_row
# array([[2, 3],
#        [4, 5],
#        [6, 7]])
```

# 11. 生成随机数

```python python
rng = np.random.default_rng()
rng.integers(5, size=(2, 4)) # 两行四列, 随机整数, 小于5
rng.random((3, 2)) # 三行两列 0-1之间 float
```

# 12. 去重和计数

```python python
a = np.array([11, 11, 12, 13, 14, 15, 16, 17, 12, 13, 11, 14, 18, 19, 20])
unique_values = np.unique(a) # 去重 [11 12 13 14 15 16 17 18 19 20]
unique_values, indices_list = np.unique(a, return_index=True) # 序号 [ 0  2  3  4  5  6  7 12 13 14]
unique_values, occurrence_count = np.unique(a, return_counts=True) # 数量 [3 2 2 2 1 1 1 1 1 1]
# 对多维数组也可用
a_2d = np.array([[1, 2, 3, 4], [5, 6, 7, 8], [9, 10, 11, 12], [1, 2, 3, 4]])
unique_values = np.unique(a_2d) # 去重 [ 1  2  3  4  5  6  7  8  9 10 11 12]
unique_rows = np.unique(a_2d, axis=0) # 获得去重的行
# [[ 1  2  3  4]
#  [ 5  6  7  8]
#  [ 9 10 11 12]]
unique_rows, indices, occurrence_count = np.unique(a_2d, axis=0, return_counts=True, return_index=True)
indices # 所得行的序号 [0 1 2]
occurrence_count # 所得行的数量 [2 1 1]
```

# 13. 矩阵转置和变形

```python python
data = np.array([1, 2, 3, 4, 5, 6])
# 变形
data.reshape(2, 3)
# array([[1, 2, 3],
#        [4, 5, 6]])
data.reshape(3, 2)
# array([[1, 2],
#        [3, 4],
#        [5, 6]])
# 转置
data = data.reshape(2, 3) # 先准备一个 23 矩阵
# array([[1, 2, 3],
#        [4, 5, 6]])
data.transpose() # 转置
# [[1 4]
#  [2 5]
#  [3 6]]
# 也可以直接用T
data.T
# [[1 4]
#  [2 5]
#  [3 6]]
```

# 14. 数组逆序

```python python
arr = np.array([1, 2, 3, 4, 5, 6, 7, 8])
reversed_arr = np.flip(arr) # [8 7 6 5 4 3 2 1]
# 二维数组逆序
arr_2d = np.array([[1, 2, 3, 4], [5, 6, 7, 8], [9, 10, 11, 12]])
reversed_arr = np.flip(arr_2d)
# [[12 11 10  9]
#  [ 8  7  6  5]
#  [ 4  3  2  1]]
# 针对的维度逆序
reversed_arr_rows = np.flip(arr_2d, axis=0)
# [[ 9 10 11 12]
#  [ 5  6  7  8]
#  [ 1  2  3  4]]
# 对切片逆序并赋值
arr_2d[:,1] = np.flip(arr_2d[:,1])
# [[ 1 10  3  4]
#  [ 8  7  6  5]
#  [ 9  2 11 12]]
```

# 15. 多维数组展开

```python python
x = np.array([[1 , 2, 3, 4], [5, 6, 7, 8], [9, 10, 11, 12]])
x.flatten() # 拷贝展开 array([ 1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11, 12])
a2 = x.ravel() # 引用展开 array([ 1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11, 12])
```

# 16. 内置文档

```python python
# help(max) 输出说明
# max? 同名所有函数说明
a = np.array([1, 2, 3, 4, 5, 6])
# a? 变量说明
```

# 17. 实现数学公式

```python python
predictions = np.array([1, 2, 3])
labels = np.array([1, 1, 1])
# 例如均方误差公式
error = (1 / 3) * np.sum(np.square(predictions - labels))
```

# 18. np对象导入导出

```python python
a = np.array([1, 2, 3, 4, 5, 6])
# np.save('filename', a) 存成文件
# b = np.load('filename.npy') 从文件读取
# 使用csv格式
# np.savetxt('new_file.csv', a)
# np.loadtxt('new_file.csv')
```

# 19. 使用 Pandas 库进行csv导入导出操作

```python python
import pandas as pd

# # If all of your columns are the same type:
# x = pd.read_csv('music.csv', header=0).values
# print(x)
# [['Billie Holiday' 'Jazz' 1300000 27000000]
#  ['Jimmie Hendrix' 'Rock' 2700000 70000000]
#  ['Miles Davis' 'Jazz' 1500000 48000000]
#  ['SIA' 'Pop' 2000000 74000000]]

# # You can also simply select the columns you need:
# x = pd.read_csv('music.csv', usecols=['Artist', 'Plays']).values
# print(x)
# [['Billie Holiday' 27000000]
#  ['Jimmie Hendrix' 70000000]
#  ['Miles Davis' 48000000]
#  ['SIA' 74000000]]
```

# 20. 使用 Matplotlib 绘制数据图像

```python python
import matplotlib.pyplot as plt
# 显示数组
a = np.array([2, 1, 5, 7, 4, 6, 8, 14, 10, 9, 18, 20, 22])
plt.plot(a)
plt.show()
# 两种数据
x = np.linspace(0, 5, 20)
y = np.linspace(0, 10, 20)
plt.plot(x, y, 'purple') # line
plt.plot(x, y, 'o')      # dots
plt.show()
# 高级使用
fig = plt.figure()
ax = fig.add_subplot(projection='3d')
X = np.arange(-5, 5, 0.15)
Y = np.arange(-5, 5, 0.15)
X, Y = np.meshgrid(X, Y)
R = np.sqrt(X**2 + Y**2)
Z = np.sin(R)

ax.plot_surface(X, Y, Z, rstride=1, cstride=1, cmap='viridis')
plt.show()
```

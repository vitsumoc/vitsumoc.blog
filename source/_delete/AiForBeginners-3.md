---
title: AI入门笔记（3）——感知器
url: AiForBeginners-3
date: 2023-12-01 09:32:11
categories: 
- AI学习
tags:
- AI
- 笔记
---

# 课程

https://github.com/microsoft/AI-For-Beginners/blob/main/lessons/3-NeuralNetworks/03-Perceptron/README.md

这是微软提供的AI-For-Beginners课程第三课，介绍了什么是感知器（Perceptron）

<!-- more -->

# 内容

感知器```Perceptron```是一种二元分类模型，总是能根据输入产生一个+1或-1的输出。

感知器进行计算时需要权重```weight```的参与，权重会导致感知器产生正确或错误的结果，训练的过程既是修改权重不断增加结果的正确率。

感知器只能解决线性分类的问题，如果一个问题无法被线性分类，感知器就不会收敛，例如异或问题。


# 随堂作业

在本课的作业中，需要使用 ```Jupyter``` 构建代码+文档的环境，使用 ```sklearn``` 创造测试数据，使用 ```numPy``` 表示和处理数据，使用 ```matpoltlib``` 绘制数据图像，使用 ```ipywidgets``` 交互式的查看训练过程。

## 训练感知器分类数据

https://github.com/vitsumoc/exercise-AI-Beginner/blob/main/3-Perceptron/perceptron.py

作业中使用代码实现了训练感知器的过程：

1. 创建数据集合，分类为训练数据和测试数据
2. 将训练数据分类为pos和neg
3. 初始化权重值
4. 设置训练次数并开始训练，每次选择随机的数据进行训练
5. 在每次训练错误时，使用本次选择的数据对权重进行调整
6. 使用测试数据验证训练后的权重值

## 感知器的局限性

https://github.com/vitsumoc/exercise-AI-Beginner/blob/main/3-Perceptron/xor.ipynb

感知器只能解决线性分类问题，对于无法使用一条直线分类的问题，往往就无法很好的收敛。

作业中的异或问题就是一个完全无法收敛的例子。

## 使用感知器 + MNIST 数据识别手写数字

https://github.com/vitsumoc/exercise-AI-Beginner/blob/main/3-Perceptron/mnist.ipynb

在这个作业中使用感知器区分手写数字图像。

使用PCA降低特征的维度，分析感知器训练结果差异的原因。

## 训练感知器识别任何手写数字

这个作业中需要拓展上一个作业的功能，训练10个不同的感知器，用来识别0-9全部的数字。

https://github.com/vitsumoc/exercise-AI-Beginner/blob/main/3-Perceptron/anyNum.ipynb

参考上一个作业的方式，训练了10组weights，来判断一个数字是或不是特定的数字。

每个感知器训练10000次，最终正确率 74.7%。
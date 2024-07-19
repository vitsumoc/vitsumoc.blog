---
title: Jupyter入门笔记
url: pythonJupyter
date: 2023-11-30 14:29:43
categories:
- python
tags:
- python
- 库
- 笔记
- 豆知识
---

# 前言

网站 https://jupyter.org/

jupyter 项目提供了可供计算的记事本，将代码、资源、交互式计算与文档结合。

<!-- more -->

# 试用

可以通过试用界面 https://jupyter.org/try 直接体验jupyter，建立大致的了解。

# 打开

```jupyter lab``` 在指定路径打开jupyter lab，程序会占用8888端口，可通过```http://localhost:8888/```访问图形化界面。

# 文件

jupyter会将执行程序的目录作为文件系统的根目录。

jupyter的文件后缀为 ```.ipynb``` 其中可以混合代码、文档、输出。

可以直接在 ```jupyter lab``` 提供的浏览器界面中新建、编辑、删除文件。

# 内容编辑

以下是一个混合了 文档、代码、输出、图像、组件的文件截图，因为导出的PDF不支持组件，所以组件输出为文本。

![](JupyterLab-1.png)

![](JupyterLab-2.png)

![](JupyterLab-3.png)

github也支持 ```.ipynb``` 格式，但同样不支持组件，这是上方图片文件的原始内容：

https://github.com/vitsumoc/exercise-AI-Beginner/blob/main/PyBeginner/jupyter.ipynb
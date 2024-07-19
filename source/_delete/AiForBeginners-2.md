---
title: AI入门笔记（2）——知识表示与专家系统
url: AiForBeginners-2
date: 2023-11-24 15:33:38
categories:
- AI学习
tags:
- AI
- 笔记
---

# 课程

https://github.com/microsoft/AI-For-Beginners/blob/main/lessons/2-Symbolic/README.md

这是微软提供的AI-For-Beginners课程第二课，介绍了过去常见的自顶向下的AI设计方法。

<!-- more -->

# 内容

通过DIKW金字塔，探讨了 数据、信息、知识、智慧 的含义，传统的人工智能实现方式就是一类尝试将数据组织成知识的方法。

探讨了使用计算机表达知识的几种方式。

## 专家系统

介绍了早期```symbolic AI```的一种成功实践：专家系统。

将专家系统的实现区分为两种类型：向后推理与向前推理。

后向推理实现专家系统的代码实践：https://github.com/vitsumoc/exercise-AI-Beginner/blob/main/2-Symbolic/animal_Inference.py

## 本体论和语义网

```ontology``` 本体指的是某个概念实体，```Semantic Web``` 语义网指的是对本体的各种规范性描述的集合，简单的有对本体属性的描述，复杂的有对各种逻辑关系的描述。

本体和语义网也是对人类思考方式的归纳和模仿，是一种组织复杂数据形成知识的方式，```WikiData``` 就是这样的一个知识库。

使用语义网实现家谱查询系统的代码实践：https://github.com/vitsumoc/exercise-AI-Beginner/blob/main/2-Symbolic/family_ontology.py

# 总结

学习了```Symbolic AI```的概念，历史，还通过几个简单例子进行了最简单的了解。

可以感受到曾经计算机行业的先驱者们为了赋予计算机智能，付出了多少辛劳和汗水，也取得了巨大的成果。

However, the important characteristics of knowledge-based systems is that you can always *explain* exactly how any of the decisions were made.
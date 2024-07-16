---
title: 工作周报可视化
url: weekreport2chart
date: 2023-12-05 11:31:51
categories:
- 小玩具
tags:
- 小玩具
- python
- js
---

# 起因

这个项目是一个纯粹的小玩具，起因是我公司的工作周报都是 ```.doc``` 格式存储的，现在到年底了，我又比较想知道我一年都干了哪些工作。显而易见的一个方式就是提取所有周报文字内容做词频分析。

<!-- more -->

# 效果

完成之后的效果还算不错，源码也放在了github上：

https://github.com/vitsumoc/weekreport2chart

提取一段时间的工作周报内容，生成词云和河流图

![](wordcloud.png)

![](river.png)

可以直接过滤低频词汇，或手动操作删除某些虚词、连词等

![](disable.png)

# 相关库

使用libreoffice将doc转为docx

使用结巴分词分词：https://github.com/fxsjy/jieba

使用wordcloudjs词云：https://wordcloud2-js.timdream.org/#love

使用echarts河流图
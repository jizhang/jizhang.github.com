---
layout: post
title: "数据挖掘指南[8]聚类"
date: 2015-01-15 12:24
comments: true
categories: [Tutorial, Translation]
published: true
---

原文：http://guidetodatamining.com/chapter-8/

前几章我们学习了如何构建分类系统，使用的是已经标记好类别的数据集进行训练：

![](https://github.com/jizhang/guidetodatamining/raw/master/img/chapter-8/chapter-8-1.png)

训练完成后我们就可以用来预测了：这个人看起来像是篮球运动员，那个人可能是练体操的；这个人三年内不会患有糖尿病。

可以看到，分类器在训练阶段就已经知道各个类别的名称了。那如果我们不知道呢？如何构建一个能够自动对数据进行分组的系统？比如有1000人，每人有20个特征，我想把这些人分为若干个组。

![](https://github.com/jizhang/guidetodatamining/raw/master/img/chapter-8/chapter-8-2.png)

这个过程叫做聚类：通过物品特征来计算距离，并自动分类到不同的群集或组中。有两种聚类算法比较常用：

**k-means聚类算法**

我们会事先告诉这个算法要将数据分成几个组，比如“请把这1000个人分成5个组”，“将这些网页分成15个组”。这种方法就叫k-means，我们会在后面的章节讨论。

## 层次聚类法

对于层次聚类法，我们不需要预先指定分类的数量，这个算方法会将每条数据都当作是一个分类，每次迭代的时候合并距离最近的两个分类，直到剩下一个分类为止。因此聚类的结果是：顶层有一个大分类，这个分类下有两个子分类，每个子分类下又有两个子分类，依此类推，层次聚类也因此得命。

![](https://github.com/jizhang/guidetodatamining/raw/master/img/chapter-8/chapter-8-3.png)

在合并的时候我们会计算两个分类之间的距离，可以采用不同的方法。如下图中的A、B、C三个分类，我们应该将哪两个分类合并起来呢？

![](https://github.com/jizhang/guidetodatamining/raw/master/img/chapter-8/chapter-8-4.png)

[前往GitHub阅读全文](https://github.com/jizhang/guidetodatamining/blob/master/chapter-8.md)

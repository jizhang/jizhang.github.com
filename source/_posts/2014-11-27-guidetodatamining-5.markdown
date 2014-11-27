---
layout: post
title: "数据挖掘指南[5]进一步探索分类"
date: 2014-11-27 12:00
comments: true
categories: [Tutorial, Translation]
published: true
---

## 效果评估算法和kNN

让我们回到上一章中运动项目的例子。

![](https://github.com/jizhang/guidetodatamining/raw/master/img/chapter-5/chapter-5-1.png)

在那个例子中，我们编写了一个分类器程序，通过运动员的身高和体重来判断她参与的运动项目——体操、田径、篮球等。

上图中的Marissa Coleman，身高6尺1寸，重160磅，我们的分类器可以正确的进行预测：

```python
>>> cl = Classifier('athletesTrainingSet.txt')
>>> cl.classify([73, 160])
'Basketball'
```

对于身高4尺9寸，90磅重的人：

```python
>>> cl.classify([59, 90])
'Gymnastics'
```

当我们构建完一个分类器后，应该问以下问题：

* 分类器的准确度如何？
* 结果理想吗？
* 如何与其它分类器做比较？

![](https://github.com/jizhang/guidetodatamining/raw/master/img/chapter-5/chapter-5-2.png)

[前往GitHub阅读全文](https://github.com/jizhang/guidetodatamining/blob/master/chapter-5.md)

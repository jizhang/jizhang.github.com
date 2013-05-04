---
layout: post
title: "Cascalog：基于Clojure的Hadoop查询语言"
date: 2013-05-01 18:01
comments: true
categories: Translation
tags: [clojure]
published: false
---

原文：[http://nathanmarz.com/blog/introducing-cascalog-a-clojure-based-query-language-for-hado.html](http://nathanmarz.com/blog/introducing-cascalog-a-clojure-based-query-language-for-hado.html)

我非常兴奋地告诉大家，[Cascalog](http://github.com/nathanmarz/cascalog)开源了！Cascalog受[Datalog](http://en.wikipedia.org/wiki/Datalog)启发，是一种基于Clojure、运行于Hadoop平台上的查询语言。

## 特点

* **简单** - 使用相同的语法编写函数、过滤规则、聚合运算；数据联合（join）变得简单而自然。
* **表达能力强** - 强大的逻辑组合条件，你可以在查询语句中任意编写Clojure函数。
* **交互性** - 可以在Clojure REPL中执行查询语句。
* **可扩展** - Cascalog的查询语句是一组MapReduce脚本。
* **任意数据源** - HDFS、数据库、本地数据、以及任何能够使用Cascading的`Tap`读取的数据。
* **正确处理空值** - 空值往往让事情变得棘手。Cascalog提供了内置的“非空变量”来自动过滤空值。
* **与Cascading结合** - 使用Cascalog定义的流程可以在Cascading中直接使用，反之亦然。
* **与Clojure结合** - 能够使用普通的Clojure函数来编写操作流程、过滤规则，又因为Cascalog是一种Clojure DSL，因此也能在其他Clojure代码中使用。

<!--more-->

好，下面就让我们开始Cascalog的学习之旅！我会用一系列的示例来介绍Cascalog。这些示例会使用到项目本身提供的“试验场”数据集。我建议你立刻下载Cascalog，一边阅读本文一边在REPL中操作。（安装启动过程只有几分钟，README中有步骤）

## 基本查询

首先让我们启动REPL，并加载“试验场”数据集：

```clojure
lein repl
user=> (use 'cascalog.playground) (bootstrap)
```

以上语句会加载本文用到的所有模块和数据。你可以阅读项目中的`playground.clj`文件来查看这些数据。下面让我们执行第一个查询语句，找出年龄为25岁的人：

```clojure
user=> (?<- (stdout) [?person] (age ?person 25))
```

这条查询语句可以这样阅读：找出所有`age`等于25的`?person`。执行过程中你可以看到Hadoop输出的日志信息，几秒钟后就能看到查询结果。

好，让我们尝试稍复杂的例子。我们来做一个范围查询，找出年龄小于30的人：

```clojure
user=> (?<- (stdout) [?person] (age ?person ?age) (< ?age 30))
```

看起来也不复杂。这条语句中，我们将人的年龄绑定到了`?age`变量中，并对该变量做出了“小于30”的限定。

我们重新执行这条语句，只是这次会将人的年龄也输出出来：

```clojure
user=> (?<- (stdout) [?person ?age] (age ?person ?age)
            (< ?age 30))
```

我们要做的仅仅是将`?age`添加到向量中去。

让我们执行另一条查询，找出艾米丽关注的所有男性：

```clojure
user=> (?<- (stdout) [?person] (follows "emily" ?person)
            (gender ?person "m"))
```

可能你没有注意到，这条语句使用了联合查询。各个数据集中的`?person`值都必须对应，而`follows`和`gender`分属于不同的数据集，Cascalog便会使用联合查询。

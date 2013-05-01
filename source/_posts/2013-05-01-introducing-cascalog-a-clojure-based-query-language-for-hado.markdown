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


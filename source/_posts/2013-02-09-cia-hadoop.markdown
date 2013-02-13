---
layout: post
title: "Clojure实战(1)：编写Hadoop MapReduce脚本"
date: 2013-02-09 16:43
comments: true
categories: Tutorial
tags: [clojure, hadoop]
published: false
---

Hadoop简介
----------

众所周知，我们已经进入了大数据时代，每天都有PB级的数据需要处理、分析，从中提取出有用的信息。Hadoop就是这一时代背景下的产物。它是Apache基金会下的开源项目，受[Google两篇论文](http://en.wikipedia.org/wiki/Apache_Hadoop#Papers)的启发，采用分布式的文件系统HDFS，以及通用的MapReduce解决方案，能够在数千台物理节点上进行分布式并行计算。

对于Hadoop的介绍这里不再赘述，读者可以[访问其官网](http://hadoop.apache.org/)，或阅读[Hadoop权威指南](http://product.dangdang.com/main/product.aspx?product_id=21127813)。

Hadoop项目是由Java语言编写的，运行在JVM之上，因此我们可以直接使用Clojure来编写MapReduce脚本，这也是本文的主题。Hadoop集群的搭建不在本文讨论范围内，而且运行MapReduce脚本也无需搭建测试环境。

clojure-hadoop类库
------------------

Hadoop提供的API是面向Java语言的，如果不想在Clojure中过多地操作Java对象，那就需要对API进行包装（wrapper），好在已经有人为我们写好了，它就是[clojure-hadoop]（https://github.com/alexott/clojure-hadoop）。

从clojure-hadoop的项目介绍中可以看到，它提供了不同级别的包装，你可以选择完全规避对Hadoop类型和对象的操作，使用纯Clojure语言来编写脚本；也可以部分使用Hadoop对象，以提升性能（因为省去了类型转换过程）。这里我们选择前一种，即完全使用Clojure语言。

示例：Wordcount
---------------

Wordcount，统计文本文件中每个单词出现的数量，可以说是数据处理领域的“Hello, world!”。这一节我们就通过它来学习如何编写MapReduce脚本。

### Leiningen 2

前几章我们使用的项目管理工具`lein`是1.7版的，而前不久Leiningen 2已经正式发布了，因此从本章开始我们的示例都会基于新版本。新版`lein`的安装过程也很简单：

```bash
$ cd ~/bin
$ wget https://raw.github.com/technomancy/leiningen/stable/bin/lein
$ chmod 755 lein
$ lein repl
user=>
```

其中，`lein repl`这一步会下载`lein`运行时需要的文件，包括Clojure 1.4。

### 新建项目

```bash
$ lein new cia-hadoop
```

编辑`project.clj`文件，添加依赖项`clojure-hadoop "1.4.1"`，尔后执行`lein deps`。

示例：统计浏览器类型
--------------------

小结
----

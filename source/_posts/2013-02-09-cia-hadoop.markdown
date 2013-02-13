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

示例：Wordcount
---------------

示例：统计浏览器类型
--------------------

小结
----

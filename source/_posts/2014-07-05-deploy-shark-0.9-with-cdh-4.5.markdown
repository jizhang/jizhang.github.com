---
layout: post
title: "在CDH 4.5上安装Shark 0.9"
date: 2014-07-05 17:16
comments: true
categories: [Notes, Big Data]
published: false
---

[Spark](http://spark.apache.org)是一个新兴的大数据计算平台，它的优势之一是内存型计算，因此对于需要多次迭代的算法尤为适用。同时，它又能够很好地融合到现有的[Hadoop](http://hadoop.apache.org)生态环境中，包括直接存取HDFS上的文件，以及运行于YARN之上。对于[Hive](http://hive.apache.org)，Spark也有相应的替代项目——[Shark](http://shark.cs.berkeley.edu/)，能做到 **drop-in replacement** ，直接构建在现有集群之上。本文就将简要阐述如何在CDH4.5上搭建Shark0.9集群。

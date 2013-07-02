---
layout: post
title: "Apache Hadoop YARN - 项目背景与简介"
date: 2013-05-25 10:57
comments: true
categories: Translation
tags: [hadoop]
published: false
---

原文：http://hortonworks.com/blog/apache-hadoop-yarn-background-and-an-overview/

日前，Apache Hadoop YARN已被提升为Apache软件基金会的子项目，这是一个值得庆祝的里程碑。这里我们也第一时间为各位献上Apache Hadoop YARN项目的系列介绍文章。YARN是一个普适的、分布式的应用管理框架，运行于Hadoop集群之上，用以替代传统的Apache Hadoop MapReduce框架。

## MapReduce 模式

本质上来说，MapReduce模型包含两个部分：一是Map过程，将数据拆分成若干份，分别处理，彼此之间没有依赖关系；二是Reduce过程，将中间结果汇总计算成最终结果。这是一种简单而又条件苛刻的模型，但也促使它成为高效和极易扩展的并行计算方式。

Apache Hadoop MapReduce是当下最流行的开源MapReduce模型。

特别地，当MapReduce配合分布式文件系统，类似Apache Hadoop HDFS，就能在大集群上提供高吞吐量的计算，这一经济效应是Hadoop得以流行的重要原因。

这一模式成功的原因之一是，它使用的是“移动计算能力至数据节点”而非通过网络“移动数据至计算节点”的方式。具体来说，一个MapReduce任务会被调度到输入数据所在的HDFS节点执行，这会极大地减少I/O支出，因为大部分I/O会发生在本地磁盘或是同一机架中——这是核心优势。



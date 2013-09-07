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

## 回顾2011年的Apache Hadoop MapReduce

Apache Hadoop MapReduce是[Apache基金会](http://www.apache.org/)下的开源项目，实现了如上所述的MapReduce编程模式。作为一个在该项目中全职开发了六年的工作者，我通常会将它细分为以下几个部分：

* 提供给最终用户使用的 **MapReduce API** ，用来编写MapReduce应用程序。
* **MapReduce框架** ，用来实现运行时的各个阶段，即map、sort/shuffle/merge、reduce。
* **MapReduce系统** ，一个完整的后端系统，用来运行用户的MapReduce应用程序，管理集群资源，调度上千个并发脚本。

这样的划分可以带来非常明显的优势，即最终用户只需关心MapReduce API，而让框架和后端系统去处理资源管理、容错、调度等细节。

目前，Apache Hadoop MapReduce系统由一个JobTracker和多个TaskTracker组成，也分别称他们为master和slave节点。

![MRArch.png](http://hortonworks.com/wp-content/uploads/2012/08/MRArch.png)

JobTracker负责的工作包括资源管理（即管理工作节点TaskTracker），跟踪资源消耗和可用情况，以及每个脚本的生命周期（脚本调度，进度跟踪，容错等）。

TaskTracker的职责比较简单：根据JobTracker的指令来启动和关闭工作进程，并定时向JobTracker汇报处理进度。

其实很早我们就意识到Hadoop的MapReduce框架需要被拆解和调整，特别是JobTracker，我们需要提升它的可扩展性，提高对集群的利用率，让用户能够方便地进行升级（即用户需要的敏捷性），并能支持MapReduce以外的脚本类型。

长久以来，我们都在做修复和更新，如近期加入的JobTracker高可用和HDFS故障恢复（这两个特性都已包含在[Hortonworks Data Platform v1](http://hortonworks.com/download/)中）。但我们渐渐发现，这些特性会增加维护成本，而且并不能解决一些核心问题，如支持非MapReduce脚本，以及敏捷性。


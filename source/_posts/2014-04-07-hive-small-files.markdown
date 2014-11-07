---
layout: post
title: "Hive小文件问题的处理"
date: 2014-04-07 17:09
comments: true
categories: [Notes, Big Data]
published: false
---

Hive的后端存储是HDFS，它对大文件的处理是非常高效的，如果合理配置文件系统的块大小，NameNode可以支持的总数据量也可以达到很高的水平。但是在数据仓库中，越是上层的表其汇总程度就越高，数据量也就越小。而且这些表通常会按日期进行分区，随着时间的推移，HDFS的文件数目就会逐渐增加。

## 小文件带来的问题

关于这个问题的阐述可以读一读Cloudera的[这篇文章](http://blog.cloudera.com/blog/2009/02/the-small-files-problem/)。简单来说，HDFS的文件元信息，包括位置、大小、分块信息等，都是保存在NameNode的内存中的。每个对象大约占用150个字节，因此一千万个文件及分块就会占用约3G的内存空间，一旦接近这个量级，NameNode的性能就会开始下降了。

此外，HDFS读写小文件时也会更加耗时，因为每次都需要从NameNode获取元信息，并与对应的DataNode建立连接。对于MapReduce程序来说，小文件还会增加Mapper的个数，每个脚本只处理很少的数据，浪费了大量的调度时间。当然，这个问题可以通过使用CombinedInputFile和JVM重用来解决。

<!-- more -->

## Hive小文件产生的原因

前面已经提到，

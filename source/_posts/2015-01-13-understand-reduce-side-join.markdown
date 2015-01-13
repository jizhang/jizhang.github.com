---
layout: post
title: "深入理解Reduce-side Join"
date: 2015-01-13 14:20
comments: true
categories: [BigData]
published: false
---

在[《MapReduce Design Patterns》][1]一书中，作者给出了Reduce-side Join的实现方法，大致步骤如下：

1. 使用[MultipleInputs][2]指定不同的来源表和相应的Mapper类；
2. Mapper输出的Key为Join的字段内容，Value为打了来源表标签的记录；
3. Reducer在接收到同一个Key的记录后，执行以下两步：
    1. 遍历Values，根据标签将来源表的记录分别放到两个List中；
    2. 遍历两个List，输出Join结果。

具体实现可以参考[这段代码][3]。但是这种实现方法有一个问题：如果同一个Key的记录数过多，存放在List中就会占用很多内存，严重的会造成内存溢出（Out of Memory, OOM）。这种方法在一对一的情况下没有问题，而一对多、多对多的情况就会有隐患。那么，Hive在做Reduce-side Join时是如何避免OOM的呢？两个关键点：

1. Reducer在遍历Values时，会将前面的表缓存在内存中，对于最后一张表则边扫描边输出；
2. 如果前面几张表内存中放不下，就写入磁盘。

<!-- more -->


[1]: http://www.amazon.com/MapReduce-Design-Patterns-Effective-Algorithms/dp/1449327176
[2]: https://hadoop.apache.org/docs/r1.0.4/api/org/apache/hadoop/mapred/lib/MultipleInputs.html
[3]: https://github.com/jizhang/mapred-sandbox/blob/master/src/main/java/com/shzhangji/mapred_sandbox/join/InnerJoinJob.java

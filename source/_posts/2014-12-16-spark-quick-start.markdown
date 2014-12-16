---
layout: post
title: "Spark快速入门"
date: 2014-12-16 15:59
comments: true
categories: [Tutorial]
published: false
---

![](http://spark.apache.org/images/spark-logo.png)

[Apache Spark](http://spark.apache.org)是新兴的一种快速通用的大规模数据处理引擎。它的优势有三个方面：

* **通用计算引擎** 能够运行MapReduce、数据挖掘、图运算、流式计算、SQL等多种框架；
* **基于内存** 数据可缓存在内存中，特别适用于需要迭代多次运算的场景；
* **与Hadoop集成** 能够直接读写HDFS中的数据，并能运行在YARN之上。

Spark是用[Scala语言](http://www.scala-lang.org/)编写的，所提供的API也很好地利用了这门语言的特性。它也可以使用Java和Python编写应用。本文将用Scala进行讲解。

## 安装Spark和SBT

* 从[官网](http://spark.apache.org/downloads.html)上下载编译好的压缩包，解压到一个文件夹中。下载时需注意对应的Hadoop版本，如要读写CDH4 HDFS中的数据，则应下载Pre-built for CDH4这个版本。
* 为了方便起见，可以将spark/bin添加到$PATH环境变量中：

```bash
export SPARK_HOME=/path/to/spark
export PATH=$PATH:$SPARK_HOME/bin
```

* 在练习例子时，我们还会用到[SBT](http://www.scala-sbt.org/)这个工具，它是用来编译打包Scala项目的。Linux下的安装过程比较简单：
    * 下载[sbt-launch.jar](https://repo.typesafe.com/typesafe/ivy-releases/org.scala-sbt/sbt-launch/0.13.7/sbt-launch.jar)到$HOME/bin目录；
    * 新建$HOME/bin/sbt文件，权限设置为755，内容如下：

```bash
SBT_OPTS="-Xms512M -Xmx1536M -Xss1M -XX:+CMSClassUnloadingEnabled -XX:MaxPermSize=256M"
java $SBT_OPTS -jar `dirname $0`/sbt-launch.jar "$@"
```

<!-- more -->

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

## 日志分析示例

假设我们有如下格式的日志文件，保存在/tmp/logs.txt文件中：

```text
2014-12-11 18:33:52	INFO	Java	some message
2014-12-11 18:34:33	INFO	MySQL	some message
2014-12-11 18:34:54	WARN	Java	some message
2014-12-11 18:35:25	WARN	Nginx	some message
2014-12-11 18:36:09	INFO	Java	some message
```

每条记录有四个字段，即时间、级别、应用、信息，使用制表符分隔。

Spark提供了一个交互式的命令行工具，可以直接执行Spark查询：

```
$ spark-shell
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 1.1.0
      /_/
Spark context available as sc.
scala>
```

### 加载并预览数据

```
scala> val lines = sc.textFile("/tmp/logs.txt")
lines: org.apache.spark.rdd.RDD[String] = /tmp/logs.txt MappedRDD[1] at textFile at <console>:12

scala> lines.first()
res0: String = 2014-12-11 18:33:52	INFO	Java	some message
```

* sc是一个SparkContext类型的变量，可以认为是Spark的入口，这个对象在spark-shell中已经自动创建了。
* sc.textFile()用于生成一个RDD，并声明该RDD指向的是/tmp/logs.txt文件。RDD可以暂时认为是一个列表，列表中的元素是一行行日志（因此是String类型）。这里的路径也可以是HDFS上的文件，如hdfs://user/hadoop/logs.txt。
* lines.first()表示调用RDD提供的一个方法：first()，返回第一行数据。

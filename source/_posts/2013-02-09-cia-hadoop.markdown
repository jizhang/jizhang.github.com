---
layout: post
title: "Clojure实战(4)：编写Hadoop MapReduce脚本"
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

### Map和Reduce

MapReduce，简称mapred，是Hadoop的核心概念之一。可以将其理解为处理问题的一种方式，即将大问题拆分成多个小问题来分析和解决，最终合并成一个结果。其中拆分的过程就是Map，合并的过程就是Reduce。

以Wordcount为例，将一段文字划分成一个个单词的过程就是Map。这个过程是可以并行执行的，即将文章拆分成多个段落，每个段落分别在不同的节点上执行划分单词的操作。这个过程结束后，我们便可以统计各个单词出现的次数，这也就是Reduce的过程。同样，Reduce也是可以并发执行的。整个过程如下图所示：

![Wordcount](/images/cia-hadoop/wordcount.png)

中间Shuffle部分的功能是将Map输出的数据按键排序，交由Reduce处理。整个过程全部由Hadoop把控，开发者只需编写`Map`和`Reduce`函数，这也是Hadoop强大之处。

#### 编写Map函数

在本示例中，我们处理的原始数据是文本文件，Hadoop会逐行读取并调用Map函数。Map函数会接收到两个参数：`key`是一个长整型，表示该行在整个文件中的偏移量，很少使用；`value`则是该行的内容。以下是将一行文字拆分成单词的Map函数：

```clojure
;; src/cia_hadoop/wordcount.clj

(ns cia-hadoop.wordcount
  (:require [clojure-hadoop.wrap :as wrap]
            [clojure-hadoop.defjob :as defjob])
  (:import [java.util StringTokenizer])
  (:use clojure-hadoop.job))

(defn my-map [key value]
  (map (fn [token] [token 1])
       (enumeration-seq (StringTokenizer. value))))
```

可以看到，这是一个纯粹的Clojure函数，并没有调用Hadoop的API。函数体虽然只有两行，但还是包含了很多知识点的：

`(map f coll)`函数的作用是将函数`f`应用到序列`coll`的每个元素上，并返回一个新的序列。如`(map inc [1 2 3])`会对每个元素做加1操作（参考`(doc inc)`），返回`[2 3 4]`。值得一提的是，`map`函数返回的是一个惰性序列（lazy sequence），即序列元素不会一次性完全生成，而是在遍历过程中逐个生成，这在处理元素较多的序列时很有优势。

`map`函数接收的参数自然不会只限于Clojure内部函数，我们可以将自己定义的函数传递给它：

```clojure
(defn my-inc [x]
  (+ x 1))

(map my-inc [1 2 3]) ; -> [2 3 4]
```

我们更可以传递一个匿名函数给`map`。上一章提过，定义匿名函数的方式是使用`fn`，另外还可使用`#(...)`简写：

```clojure
(map (fn [x] (+ x 1)) [1 2 3])
(map #(+ % 1) [1 2 3])
```

对于含有多个参数的情况：

```clojure
((fn [x y] (+ x y)) 1 2) ; -> 3
(#(+ %1 %2) 1 2) ; -> 3
```

`my-map`中的`(fn [token] [token 1])`即表示接收参数`token`，返回一个向量`[token 1]`，其作用等价于`#(vector % 1)`。为何是`[token 1]`，是因为Hadoop的数据传输都是以键值对的形式进行的，如`["apple" 1]`即表示“apple”这个单词出现一次。

[StringTokenizer](http://docs.oracle.com/javase/6/docs/api/java/util/StringTokenizer.html)则是用来将一行文字按空格拆分成单词的。他的返回值是`Enumeration`类型，Clojure提供了`enumeration-seq`函数，可以将其转换成序列进行操作。

所以最终`my-map`函数的作用就是：将一行文字按空格拆分成单词，返回一个形如`[["apple" 1] ["orange" 1] ...]`的序列。

#### 编写Reduce函数

从上文的图表中可以看到，Map函数处理完成后，Hadoop会对结果按照键进行排序，并使用`key, [value1 value2 ...]`的形式调用Reduce函数。在clojure-hadoop中，Reduce函数的第二个参数是一个函数，其返回结果才是值的序列：

```clojure
(defn my-reduce [key values-fn]
  [[key (reduce + (values-fn))]])
```

和Map函数相同，Reduce函数的返回值也是一个序列，其元素是一个个`[key value]`。注意，函数体中的`(reduce f coll)`是Clojure的内置函数，其作用是：取`coll`序列的第1、2个元素作为参数执行函数`f`，将结果和`coll`序列的第3个元素作为参数执行函数`f`，依次类推。因此`(reduce + [1 2 3])`等价于`(+ (+ 1 2) 3)`。

#### 定义脚本

有了Map和Reduce函数，我们就可以定义一个完整的脚本了：

```clojure
(defjob/defjob job
  :map my-map
  :map-reader wrap/int-string-map-reader
  :reduce my-reduce
  :input-format :text
  :output-format :text
  :compress-output false
  :replace true
  :input "README.md"
  :output "out-wordcount")
```

简单说明一下这些配置参数：`:map`和`:reduce`分别指定Map和Reduce函数；`map-reader`表示读取数据文件时采用键为`int`、值为`string`的形式；`:input-format`至`compress-output`指定了输入输出的文件格式，这里采用非压缩的文本形式，方便阅览；`:replace`表示每次执行时覆盖上一次的结果；`:input`和`:output`则是输入的文件和输出的目录。

#### 执行脚本

我们可以采用Clojure的测试功能来执行脚本：

```clojure
;; test/cia_hadoop/wordcount_test.clj

(ns cia-hadoop.wordcount-test
  (:use clojure.test
        clojure-hadoop.job
        cia-hadoop.wordcount))

(deftest test-wordcount
  (is (run job)))
```

尔后执行：

```bash
$ lein test cia-hadoop.wordcount-test
...
13/02/14 00:25:52 INFO mapred.JobClient:  map 0% reduce 0%
..
13/02/14 00:25:58 INFO mapred.JobClient:  map 100% reduce 100%
...
$ cat out-wordcount/part-r-00000
...
"java"  1
"lein"	3
"locally"	2
"on"	1
...
```

如果想要将MapReduce脚本放到Hadoop集群中执行，可以采用以下命令：

```bash
$ lein uberjar
$ hadoop jar target/cia-hadoop-0.1.0-SNAPSHOT-standalone.jar clojure_hadoop.job -job cia-hadoop.wordcount/job
```

示例：统计浏览器类型
--------------------

小结
----

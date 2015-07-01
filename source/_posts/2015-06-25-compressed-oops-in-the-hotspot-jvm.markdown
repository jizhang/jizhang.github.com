---
layout: post
title: "HotSpot JVM中的对象指针压缩"
date: 2015-06-25 17:41
comments: true
categories: [Translation]
published: false
---

原文：https://wiki.openjdk.java.net/display/HotSpot/CompressedOops

## 什么是一般对象指针？

一般对象指针（oop, ordinary object pointer）是HotSpot虚拟机的一个术语，表示受托管的对象指针。它的大小通常和本地指针是一样的。Java应用程序和GC子系统会非常小心地跟踪这些受托管的指针，以便在销毁对象时回收内存空间，或是在对空间进行整理时移动（复制）对象。

在一些从Smalltalk和Self演变而来的虚拟机实现中都有一般对象指针这个术语，包括：

* [Self][1]：一门基于原型的语言，是Smalltalk的近亲
* [Strongtalk][2]：Smalltalk的一种实现
* [Hotspot][3]
* [V8][4]

部分系统中会使用小整型（smi, small integers）这个名称，表示一个指向30位整型的虚拟指针。这个术语在Smalltalk的V8实现中也可以看到。

## 为什么需要压缩？

在[LP64][5]系统中，指针需要使用64位来表示；[ILP32][5]系统中则只需要32位。在ILP32系统中，堆内存的大小只能支持到4Gb，这对很多应用程序来说是不够的。在LP64系统中，所有应用程序运行时占用的空间都会比ILP32大1.5倍左右，这是因为指针占用的空间增加了。虽然内存是比较廉价的，但网络带宽和缓存容量是紧张的。所以，为了解决4Gb的限制而增加堆内存的占用空间，就有些得不偿失了。

在x86芯片中，ILP32模式可用的寄存器数量是LP64模式的一半。SPARC没有此限制；RISC芯片本来就提供了很多寄存器，LP64模式下会提供更多。

压缩后的一般对象指针在使用时需要将32位整型按因数8进行扩展，并加到一个64位的基础地址上，从而找到所指向的对象。这种方法可以表示四十亿个对象，相当于32Gb的堆内存。同时，使用此法压缩数据结构也能达到和ILP32系统相近的效果。

我们使用*解码*来表示从32位对象指针转换成64位地址的过程，其反过程则称为*编码*。

<!-- more -->


[1]: https://github.com/russellallen/self/blob/master/vm/src/any/objects/oop.hh
[2]: http://code.google.com/p/strongtalk/wiki/VMTypesForSmalltalkObjects
[3]: http://hg.openjdk.java.net/hsx/hotspot-main/hotspot/file/0/src/share/vm/oops/oop.hpp
[4]: http://code.google.com/p/v8/source/browse/trunk/src/objects.h
[5]: http://docs.oracle.com/cd/E19620-01/805-3024/lp64-1/index.html

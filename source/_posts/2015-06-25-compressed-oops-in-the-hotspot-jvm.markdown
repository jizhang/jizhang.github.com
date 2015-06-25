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

<!-- more -->

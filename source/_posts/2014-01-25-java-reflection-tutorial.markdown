---
layout: post
title: "Java反射机制"
date: 2014-01-25 09:42
comments: true
categories: [Translation, Tutorial]
---

什么是反射？它有何用处？

## 1. 什么是反射？

“反射（Reflection）能够让运行于JVM中的程序检测和修改运行时的行为。”这个概念常常会和内省（Introspection）混淆，以下是这两个术语在Wikipedia中的解释：

1. 内省用于在运行时检测某个对象的类型和其包含的属性；
2. 反射用于在运行时检测和修改某个对象的结构及其行为。

从他们的定义可以看出，内省是反射的一个子集。有些语言支持内省，但并不支持反射，如C++。

![反射和内省](http://www.programcreek.com/wp-content/uploads/2013/09/reflection-introspection-650x222.png)

<!-- more -->

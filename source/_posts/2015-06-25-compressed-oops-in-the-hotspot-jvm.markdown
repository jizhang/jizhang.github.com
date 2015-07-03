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

## 什么情况下会进行压缩？

运行在ILP32模式下的Java虚拟机，或在运行时将`UseCompressedOops`标志位关闭，则所有的对象指针都不会被压缩。

如果`UseCompressedOops`是打开的，则以下对象的指针会被压缩：

* 所有对象的[klass][6]属性；
* 所有[对象指针实例][7]的属性；
* 所有对象指针数组的元素（objArray）。

Hotspot VM中，用于表示Java类的数据结构是不会压缩的，这部分数据都存放在永久代（PermGen）中。

在解释器中，一般对象指针也是不压缩的，包括JVM本地变量和栈内元素、调用参数、返回值等。解释器会在读取堆内对象时解码对象指针，并在存入时进行编码。

同样，方法调用序列（method calling sequence），无论是解释执行还是编译执行，都不会使用对象指针压缩。

在编译后的代码中，对象指针是否压缩取决于不同的优化结果。优化后的代码可能会将压缩后的对象指针直接从一处搬往另一处，而不进行解码编码操作。如果芯片（如x86）支持解码，那在使用对象指针时就不需要自行解码了。

所以，以下数据结构在编译后的代码中既可以是压缩后的对象指针，也可能是本地地址：

* 寄存器或溢出槽（spill slot）中的数据
* 对象指针映射表（GC映射表）
* 调试信息
* 嵌套在机器码中的对象指针（在非RISC芯片中支持，如x86）
* [nmethod][8]常量区（包括那些影响到机器码的重定位操作）

在HotSpot JVM的C++代码部分，对象指针压缩与否反映在C++的静态类型系统中。通常情况下，对象指针是不压缩的。具体来说，C++的成员函数在操作本地代码传递过来的指针时（如*this*），其执行过程不会有什么不同。JVM中的部分方法则提供了重载，能够处理压缩和不压缩的对象指针。

重要的C++数据不会被压缩：

* C++对象指针（*this*）
* 受托管指针的句柄（Handle类型等）
* JNI句柄（jobject类型）

C++在使用对象指针压缩时（加载和存储等），会以`narrowOop`作为标记。

## 使用压缩寻址

以下是使用对象指针压缩的x86指令示例：

```text
! int R8; oop[] R9;  // R9是64位
! oop R10 = R9[R8];  // R10是32位
! 从原始基址指针加载压缩对象指针：
movl R10, [R9 + R8<<3 + 16]
! klassOop R11 = R10._klass;  // R11是32位
! void* const R12 = GetHeapBase();
! 从压缩基址指针加载klass指针：
movl R11, [R12 + R10<<3 + 8]
```

以下sparc指令用于解压对象指针（可为空）：

```text
! java.lang.Thread::getThreadGroup@1 (line 1072)
! L1 = L7.group
ld  [ %l7 + 0x44 ], %l1
! L3 = decode(L1)
cmp  %l1, 0
sllx  %l1, 3, %l3
brnz,a   %l3, .+8
add  %l3, %g6, %l3  ! %g6是常量堆基址
```

*输出中的注解来自[PrintAssembly插件][9]。*

[1]: https://github.com/russellallen/self/blob/master/vm/src/any/objects/oop.hh
[2]: http://code.google.com/p/strongtalk/wiki/VMTypesForSmalltalkObjects
[3]: http://hg.openjdk.java.net/hsx/hotspot-main/hotspot/file/0/src/share/vm/oops/oop.hpp
[4]: http://code.google.com/p/v8/source/browse/trunk/src/objects.h
[5]: http://docs.oracle.com/cd/E19620-01/805-3024/lp64-1/index.html
[6]: http://stackoverflow.com/questions/16721021/what-is-klass-klassklass
[7]: http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7-b147/sun/jvm/hotspot/oops/Oop.java#Oop
[8]: http://openjdk.java.net/groups/hotspot/docs/HotSpotGlossary.html#nmethod
[9]: https://wiki.openjdk.java.net/display/HotSpot/PrintAssembly

---
layout: post
title: "柯里化与偏应用"
date: 2013-03-07 20:59
comments: true
categories: Translation
tags: [fp, javascript]
published: false
---

原文：[http://raganwald.com/2013/03/07/currying-and-partial-application.html](http://raganwald.com/2013/03/07/currying-and-partial-application.html)
作者：[Reginald Braithwaite](http://raganwald.com/)

上周末我参加了[wroc_love.rb大会](http://wrocloverb.com/)，其间[Steve Klabnik](http://steveklabnik.com/)的一张PPT中提到了[偏应用（Partial Application）](https://en.wikipedia.org/wiki/Partial_application)和[柯里化（Currying）](https://en.wikipedia.org/wiki/Currying)，并说这两者之间的区别如今已经不重要了。但是我不这么认为。

在这周发布的博文中，我用五种方式对`this`和闭包做了解释，但只有三到四种提到了柯里化。所以这篇博文就重点来谈谈这个。

函数参数的个数
--------------

在讲解之前，我们先明确一些术语。函数定义时会写明它所接收的参数个数（Arity）。“一元函数”（Unary）接收一个参数，“多元函数”（Polyadic）接收多个参数。还有一些特殊的名称，如“二元函数”（Binary）接收两个参数，“三元函数”（Ternary）接收三个参数等。你可以对照希腊语或拉丁语词汇来创造这些特殊的名称。

有些函数能够接收不定数量的参数，我们称之为“可变参数函数”（Variadic）。不过这类函数、以及不接收参数的函数并不是本文讨论的重点。

<!-- more -->


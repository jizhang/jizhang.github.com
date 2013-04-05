---
layout: post
title: "柯里化与偏应用（JavaScript描述）"
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

偏应用
------

偏应用的概念很容易理解，我们可以使用加法函数来做简单的演示，但如果你不介意的话，我想引用[allong.es](http://allong.es/)这一JavaScript类库中的代码来做演示，而且它也是会在日常开发中用到的代码。

作为铺垫，我们首先实现一个`map`函数，用来将某个函数应用到数组的每个元素上：

```javascript
var __map = [].map;

function map (list, unaryFn) {
  return __map.call(list, unaryFn);
};

function square (n) {
  return n * n;
};

map([1, 2, 3], square);
  //=> [1, 4, 9]
```

显然，`map`是二元函数，`square`是一元函数。当我们使用`[1, 2, 3]`和`square`作为参数来调用`map`时，我们是将这两个参数 *应用（Apply）* 到`map`函数，并获得结果。

由于`map`函数接收两个参数，我们也提供了两个参数，所以说这是一次 *完整应用* 。那何谓偏应用（或部分应用）呢？即提供少于指定数量的参数。如，仅提供一个参数来调用`map`。

如果我们只提供一个参数来调用`map`会怎么样？我们无法得到所要的结果，只能得到一个新的一元函数，通过调用这个函数并传递缺失的参数后，才能获得结果。

假设现在我们只提供一个参数给`map`，这个参数是`unaryFn`。我们从后往前来逐步实现，首先为`map`函数创建一个包装函数：

```javascript
function mapWrapper (list, unaryFn) {
  return map(list, unaryFn);
};
```

然后，我们将这个二元函数分割成两个嵌套的一元函数：

```javascript
function mapWrapper (unaryFn) {
  return function (list) {
    return map(list, unaryFn);
  };
};
```

这样一来，我们就能每次仅传递一个参数来进行调用了：

```javascript
mapWrapper(square)([1, 2, 3]);
  //=> [1, 4, 9]
```

和之前的`map`函数相较，新的函数`mapWrapper`是一元函数，它的返回值是另一个一元函数，需要再次调用它才能获得返回值。那么偏应用要从何体现？让我们从第二个一元函数着手：

```javascript
var squareAll = mapWrapper(square);
  //=> [function]

squareAll([1, 2, 3]);
  //=> [1, 4, 9]
squareAll([5, 7, 5]);
  //=> [25, 49, 25]
```

我们首先将`square`这个参数部分应用到了`map`函数，并获得一个一元函数`squareAll`，它能实现我们需要的功能。偏应用后的`map`函数十分便捷，而[allong.es](http://allong.es/)库中提供的`splat`函数做的也是相同的事情。

如果每次想要使用偏应用都需要手动编写这样一个包装函数，程序员显然会想到要自动化实现它。这就是下一节的内容：柯里化。

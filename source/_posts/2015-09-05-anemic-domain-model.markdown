---
layout: post
title: "贫血领域模型"
date: 2015-09-05 19:02
comments: true
categories: [Translation]
published: false
---

原文：http://www.martinfowler.com/bliki/AnemicDomainModel.html

贫血领域模型是一个存在已久的反模式，目前仍有许多拥趸者。一次我和Eric Evans聊天谈到它时，都觉得这个模型似乎越来越流行了。作为[领域模型](http://martinfowler.com/eaaCatalog/domainModel.html)的推广者，我们觉得这不是一件好事。

贫血领域模型的最初症状是：它第一眼看起来还真像这么回事儿。项目中有许多对象，它们的命名都是根据领域来的。对象之间有着丰富的连接方式，和真正的领域模型非常相似。但当你检视这些对象的行为时，会发现它们基本上没有任何行为，仅仅是一堆getter和setter的集合。其实这些对象在设计之初就被定义为只能包含数据，不能加入领域逻辑。这些逻辑要全部写入一组叫Service的对象中。这些Service构建在领域模型之上，使用这些模型来传递数据。

这种反模式的恐怖之处在于，它完全是和面向对象设计背道而驰。面向对象设计主张将数据和行为绑定在一起，而贫血领域模型则更像是一种面向过程设计，我和Eric在Smalltalk时就极力反对这种做法。更糟糕的时，很多人认为这些贫血领域对象是真正的对象，从而彻底误解了面向对象设计的涵义。

<!-- more -->

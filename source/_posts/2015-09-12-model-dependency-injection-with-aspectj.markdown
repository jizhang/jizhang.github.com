---
layout: post
title: "使用AspectJ向领域模型注入依赖"
date: 2015-09-12 22:03
comments: true
categories: [Notes]
published: false
---

在[贫血领域模型]({% post_url 2015-09-05-anemic-domain-model %})这篇译文中，Martin阐述了这种“反模式”的症状和问题，并引用了领域驱动设计中的话来说明领域模型和分层设计之间的关系。对于Spring项目的开发人员来说，贫血领域模型十分常见：模型（或实体）仅仅包含对数据表的映射，通常是一组私有属性和公有getter/setter，所有的业务逻辑都写在服务层中，领域模型仅仅用来传递数据。为了编写真正的领域模型，我们需要将业务逻辑移至模型对象中，这就引出另一个问题：业务逻辑通常需要调用其他服务或模型，而使用`new`关键字或由JPA创建的对象是不受Spring托管的，也就无法进行依赖注入。本文将要阐述的是如何使用面向切面编程来解决这个问题。

## 面向切面编程

面向切面编程，或[AOP](https://en.wikipedia.org/wiki/Aspect-oriented_programming)，是一种编程范式，和面向对象编程（[OOP](https://en.wikipedia.org/wiki/Object-oriented_programming)）互为补充。

<!-- more -->

## 其它方案

## 参考资料

* http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#aop-atconfigurable
* http://blog.igorstoyanov.com/2005/12/dependency-injection-or-service.html

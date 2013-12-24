---
layout: post
title: "Spring JDBC与存储过程"
date: 2013-12-24 11:00
comments: true
categories: Notes
published: false
---

[存储过程](http://baike.baidu.com/view/68525.htm)是指保存在数据库管理系统中的一组代码，主要由SQL组成，不同的数据库产品也会提供更为丰富的语法，如Oracle的[PL/SQL](http://en.wikipedia.org/wiki/PL/SQL)。存储过程的优点是能够将复杂的业务逻辑封装起来，便于重用，且因为其语句是预编译过的，执行速度也会提升。

JDBC对存储过程有全面的支持，Spring Framework中也有一系列类库可以用来操作存储过程。本文将列举这些操作，包括存储过程的传入传出参数、多结果集等应用。

## 使用纯JDBC方式调用存储过程


<!-- more -->

## 使用JdbcTemplate


## 使用SimpleJdbcCall


## 使用StoredProcedure抽象类

---
layout: post
title: "Java反射机制"
date: 2014-01-25 09:42
comments: true
categories: [Translation, Tutorial]
published: false
---

什么是反射？它有何用处？

## 1. 什么是反射？

“反射（Reflection）能够让运行于JVM中的程序检测和修改运行时的行为。”这个概念常常会和内省（Introspection）混淆，以下是这两个术语在Wikipedia中的解释：

1. 内省用于在运行时检测某个对象的类型和其包含的属性；
2. 反射用于在运行时检测和修改某个对象的结构及其行为。

从他们的定义可以看出，内省是反射的一个子集。有些语言支持内省，但并不支持反射，如C++。

![反射和内省](http://www.programcreek.com/wp-content/uploads/2013/09/reflection-introspection-650x222.png)

<!-- more -->

内省示例：`instanceof`运算符用于检测某个对象是否属于特定的类。

```java
if (obj instanceof Dog) {
    Dog d = (Dog) obj;
    d.bark();
}
```

反射示例：`Class.forName()`方法可以通过类或接口的名称（一个字符串或完全限定名）来获取对应的`Class`对象。`forName`方法会触发类的初始化。

```java
// 使用反射
Class<?> c = Class.forName("classpath.and.classname");
Object dog = c.newInstance();
Method m = c.getDeclaredMethod("bark", new Class<?>[0]);
m.invoke(dog);
```

在Java中，反射更接近于内省，因为你无法改变一个对象的结构。虽然一些API可以用来修改方法和属性的可见性，但并不能修改结构。

## 2. 我们为何需要反射？

反射能够让我们：

* 在运行时检测对象的类型；
* 动态构造某个类的对象；
* 检测类的属性和方法；
* 任意调用对象的方法；
* 修改构造函数、方法、属性的可见性；
* 以及其他

反射是框架中常用的方法。

例如，[JUnit](http://www.programcreek.com/2012/02/junit-tutorial-2-annotations/)通过反射来遍历包含 *@Test* 注解的方法，并在运行单元测试时调用它们。（[这个连接](http://www.programcreek.com/2012/02/junit-tutorial-2-annotations/)中包含了一些JUnit的使用案例）

对于Web框架，开发人员在配置文件中定义他们对各种接口和类的实现。通过反射机制，框架能够快速地动态初始化所需要的类。

例如，Spring框架使用如下的配置文件：

```xml
<bean id="someID" class="com.programcreek.Foo">
    <property name="someField" value="someValue" />
</bean>
```

当Spring容器处理&lt;bean&gt;元素时，会使用`Class.forName("com.programcreek.Foo")`来初始化这个类，并再次使用反射获取&lt;property&gt;元素对应的`setter`方法，为对象的属性赋值。

Servlet也会使用相同的机制：

```xml
<servlet>
    <servlet-name>someServlet</servlet-name>
    <servlet-class>com.programcreek.WhyReflectionServlet</servlet-class>
<servlet>
```

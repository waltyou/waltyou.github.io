---
layout: post
title: 《Effective Java》学习日志（一）：对象的创建与销毁
date: 2018-07-22 10:11:04
author: admin
comments: true
categories: [Java]
tags: [Java，Effective Java]
---

该如何编写有效的 Java 代码呢？来学习一下《Effective Java》第三版。

<!-- more -->
---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---
## 目录
{:.no_toc}

* 目录
{:toc}

---

# 全书简介

这本书是为了帮助我们有效的使用 Java 语言和它的基础库（如java.lang , java.util , java.io） 和子包（如：java.util.concurrent and java.util.function）

共分为 11 章节和 90 个 item。 

每个 Item 表示一条规则，它们可以交叉阅读，因为它们都是独立的部分。

全书脑图如下：

[![](/images/posts/Effective+Java+3rd+Edition.png)](/images/posts/Effective+Java+3rd+Edition.png)

首先来看第一章：对象的创建与销毁。

--- 

# Item 1: 考虑使用静态工厂方法来代替构造方法

好处如下：

## 1. 静态工厂方法有名字

当构造函数的参数本身不能很好的描述函数返回的是什么样的对象时，一个有好名字的静态方法，会帮助客户端代码更好的理解。

举个例子就是：构造函数 BigInteger(int, int, Random)，它返回了一个可能是素数的 BigIntege， 但是如果使用静态工厂方法 BigInteger.probablePrime ，表达就会更加清晰。

另外，我们都知道对于给定的一个标识，一个类只能有一个对应的构造函数。但有时候，为了打破这个限制，程序员可能会使用两个仅仅参数顺序不一致的构造函数来解决这个问题。这是个很不好的行为。因为使用者很可能分不清哪个构造函数该被使用，从而导致错误发生。除非他们认真的阅读使用文档。

但是静态工厂方法的名字就解决了上述问题，只需要取两个定义清晰且不同的名字就可以了。
 
## 2. 静态工厂方法不需要每次都创建新对象

这个特点允许不变类（immutable class）来使用提前构造好的实例，或来缓存他们构造的实例，又或可以重复分发已有实例来避免创建重复的对象。

举个例子就是 Boolean.valueOf(boolean)：

```java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```
这个方法从来不会创建一个对象，有点像是设计模式中的享元模式(Flyweight Pattern)。如果经常请求同样的对象，它可以极大地提高性能，特别是它们的创建代价很昂贵时。
 
静态工厂方法保证了在反复的调用中都能返回相同的对象，它的这种能力保证了类对存在的实例进行严格的控制。这种控制叫做“实例控制 instance-controlled”。

有以下几个原因来写实例控制的类：
- 保证类是单例或者不可实例化的
- 对于不变值的类，可以保证他们是相等的
- 这是享元模式的基础


---

# 未完待续.....
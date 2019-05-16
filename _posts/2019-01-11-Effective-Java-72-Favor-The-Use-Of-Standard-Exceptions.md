---
layout: post
title: 《Effective Java》学习日志（九）72:偏爱使用标准的异常
date: 2019-01-11 11:22:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]

---



<!-- more -->

------

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

------




* 目录
{:toc}

------

区分专家程序员与经验不足的程序员的一个属性是专家努力并且通常实现高度的代码重用。 Exception的重用无疑也是一件好事。 Java库提供了一组异常，涵盖了大多数API的大多数异常抛出需求。

重用标准例外有几个好处。 其中最主要的是它使您的API更易于学习和使用，因为它符合程序员已经熟悉的既定惯例。 紧接其后的是，使用您的API的程序更容易阅读，因为它们不会被不熟悉的异常所混淆。 最后（和最少），更少的异常类意味着更小的内存占用和更少的加载类所花费的时间。

最常用的异常类型是IllegalArgumentException（Item 49）。 当调用者传入一个值不合适的参数时，这通常是抛出的异常。 例如，如果调用者在表示某个操作要重复的次数的参数中传递了一个负数，则抛出此异常。

另一个常用的异常是IllegalStateException。 如果由于接收对象的状态而调用是非法的，则通常会抛出异常。 例如，如果调用者在正确初始化之前尝试使用某个对象，则抛出此异常。

可以说，每个错误的方法调用都归结为非法的参数或状态，但其他例外情况则标准地用于某些非法论证和状态。如果调用者在某些禁止空值的参数中传递null，则约定表示抛出NullPointerException而不是IllegalArgumentException。类似地，如果调用者将表示索引的参数中的超出范围的值传递给序列，则应抛出IndexOutOfBoundsException而不是IllegalArgumentException。


另一个可重用的异常是ConcurrentModificationException。如果设计为由单个线程（或外部同步）使用的对象检测到它正在被同时修改，则应该抛出它。此异常充其量只是一个提示，因为无法可靠地检测并发修改。

注意的最后一个标准例外是UnsupportedOperationException。如果对象不支持尝试的操作，则抛出异常。它的使用很少见，因为大多数对象都支持它们的所有方法。未能实现由其实现的接口定义的一个或多个可选操作的类使用此异常。例如，如果有人试图从列表中删除元素，则仅附加的List实现会抛出此异常。

**不要直接重用Exception，RuntimeException，Throwable或Error**。 将这些类看作是抽象的。 您无法可靠地测试这些异常，因为它们是方法可能引发的其他异常的超类。

此表总结了最常用的异常情况：

[![](/images/posts/effective-java-72.png)](/images/posts/effective-java-72.png)


虽然这些是迄今为止最常用的例外情况，但其他情况可能会在情况允许的情况下重复使用。例如，如果要实现算术对象（如复数或有理数），则重用ArithmeticException和NumberFormatException是合适的。如果异常符合您的需求，请继续使用它，但前提是您抛出它的条件与异常文档一致：重用必须基于文档化的语义，而不仅仅是名称。另外，如果要添加更多细节（第75项），请随意为标准异常创建子类，但请记住异常是可序列化的（第12章）。仅此一点就是没有充分理由不编写自己的异常类的理由。

选择要重用的异常可能很棘手，因为上表中的“使用场合”似乎并不相互排斥。考虑一个代表一副牌的对象的情况，并假设有一种方法来处理来自牌组的牌，该牌作为牌的大小。如果调用者传递的值大于牌组中剩余的牌数，则可以将其解释为IllegalArgumentException（handSize参数值太高）或IllegalStateException（牌组包含的牌数太少）。**在这些情况下，如果没有参数值可行，则规则是抛出IllegalStateException，否则抛出IllegalArgumentException。**




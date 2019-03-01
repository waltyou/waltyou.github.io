---
layout: post
title: 《HeadFirst设计模式》学习日志（七）:适配器与外观模式
date: 2019-02-28 11:03:00
author: admin
comments: true
categories: [Java]
tags: [Java,Design Patterns]
---



<!-- more -->

------

学习资料主要参考： 《Head First 设计模式》

------

## 目录
{:.no_toc}

* 目录
{:toc}
------

# 适配器模式

回忆装饰者模式，我们将对象包装起来，赋予它新的职责。而现在的目的是：让它们的接口看起来不像自己而像是别的东西。

## 步骤

- 客户对适配器发出请求
- 适配器把请求转换为一个或者多个被适配者接口
- 客户接收到调用结果

## 定义

将一个类的接口，转换成客户期望的另一个接口。适配器让原本接口不兼容的类可以合作无间。

类图：

[![](/images/posts/adapter-pattern.png)](/images/posts/adapter-pattern.png)

优点：使用对象组合，以修改的接口包装被适配者。这样子还有其他的好处，那就是所有被适配者的任何子类，都可以搭配适配器使用。

## 两种适配器

其实分为“对象”适配器和“类”适配器。

类的适配器，需要多重继承才能够实现，但是 Java 没有这个功能。

## 对比装饰者

装饰者的工作全部都是和“责任”相关的。一旦涉及到装饰者，就表示有一些新的行为或者责任要加入到你的设计中。

适配器的工作就是转换接口。

# 外观模式

它改变接口的原因是为了**简化接口**，它将一个或者数个类的复杂的一切都隐藏在背后，只露出一个干净美好的**外观**。

除了提供一个简洁的接口给客户端外，外观模式也可以将客户端与子系统解耦。

## 定义

提供了一个统一的接口，用来访问子系统中的一群接口。外观定义了一个高层接口，让子系统更容易使用。

# 最少知识原则

> 只和你的密友谈话。

要减少对象之间的交互，只留下几个“密友”。

## 怎么做？

在该对象的方法内，我们应该只调用以下范围的方法：

- 该对象本身
- 被当作方法的参数而传递进来的对象
- 此方法所创建或实例化的任何对象
- 对象的任何组件

避免如下连续调用方法：

```java
station.getThermoneter().getTemperature();
```

## 优缺点

优点：减少对象之间的依赖。

缺点：增加了复杂度和开发时间，并降低了运行性能。
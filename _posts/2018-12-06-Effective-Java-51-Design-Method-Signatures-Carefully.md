---
layout: post
title: 《Effective Java》学习日志（七）51：小心地设计方法签名
date: 2018-12-06 18:23:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

这个Item是一个API设计提示的抓包，本身虽然不值得为一个Item。

但是，它们将有助于使您的API更易于学习和使用，并且让代码不易出错。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---

## 目录
{:.no_toc}

* 目录
{:toc}
---



# 仔细选择方法名称

应始终遵守标准命名约定（第68项）。

您的主要目标应该是选择可理解且与同一包中的其他名称一致的名称。

您的次要目标应该是选择与更广泛的共识一致的名称。

避免使用长方法名称。如有疑问，请查看Java库API以获取指导。虽然存在许多不一致 - 不可避免的，考虑到这些图书馆的规模和范围 - 也有相当多的共识。

# 不要过分提供便利方法

每种方法都应该“拉动它的重量。”

太多的方法使得Class难以学习，使用，记录，测试和维护。

对于接口而言，这是双重的，因为太多的方法使实现者和用户的生活变得复杂。

对于您的类或接口支持的每个操作，请提供功能完备的方法。只有在经常使用它时才考虑提供“速记”。如有疑问，请将其删除。

# 避免使用长参数列表

针对四个参数或更少的参数。

大多数程序员都记不起更长的参数列表。如果您的许多方法超出此限制，则在未经常引用其文档的情况下，您的API将无法使用。现代IDE有所帮助，但您使用短参数列表仍然会更好。**长序列的相同类型的参数尤其有害**。用户不仅不能记住参数的顺序，而且当他们意外地转换参数时，他们的程序仍然可以编译和运行。他们只是不会做他们的作者的意图。

有三种技术可以缩短过长的参数列表。 

### 分解方法

一种方法是将方法分解为多种方法，每种方法只需要一部分参数。 如果不小心完成，这可能导致太多方法，但它也可以通过增加正交性来帮助减少方法计数。 

例如，考虑java.util.List接口。 它没有提供查找子列表中元素的第一个或最后一个索引的方法，这两个索引都需要三个参数。 相反，它提供了subList方法，该方法接受两个参数并返回子列表的视图。 此方法可以与indexOf或lastIndexOf方法结合使用，每个方法都有一个参数，以产生所需的功能。 此外，subList方法可以与在List实例上操作的任何方法组合，以对子列表执行任意计算。 得到的API具有非常高的功率重量比。

### 创建辅助类

缩短长参数列表的第二种技术是创建辅助类来保存参数组。 通常，这些辅助类是静态成员类（第24项）。 如果看到频繁出现的参数序列代表某个不同的实体，则建议使用此技术。 

例如，假设您正在编写一个代表纸牌游戏的类，并且您发现自己经常传递一系列两个参数来表示卡的等级及其套装。 如果添加了一个辅助类来表示一个卡并用一个辅助类的参数替换每个参数序列，那么您的API以及类的内部结构可能会受益。

### Builder 模式

结合前两个方面的第三种技术是使Builder模式（第2项）从对象构造适应方法调用。 如果你有一个包含许多参数的方法，特别是如果它们中的一些是可选的，那么定义一个代表所有参数的对象并允许客户端在这个对象上进行多次“setter”调用是有益的。 设置单个参数或小的相关组。 一旦设置了所需的参数，客户端就会调用对象的“执行”方法，该方法对参数进行任何最终有效性检查并执行实际计算。

# 对于参数类型，接口优先于类

如果有适当的接口来定义参数，请使用它来支持实现接口的类。 

例如，没有理由编写一个在输入使用Map上使用HashMap的方法。 这使您可以传入HashMap，TreeMap，ConcurrentHashMap，TreeMap的子图或任何尚未编写的Map实现。 通过使用类而不是接口，可以将客户端限制为特定实现，并在输入数据恰好以其他形式存在时强制执行不必要且可能很昂贵的复制操作。

# 首选两元素枚举类型为布尔参数

除非从方法名称中布尔值的含义很清楚，否则首选两元素枚举类型为布尔参数。 

枚举使您的代码更易于阅读和编写。 此外，它们可以让您以后轻松添加更多选项。

例如，您可能有一个具有静态工厂的温度计类型，该工厂采用此枚举：

    public enum TemperatureScale { FAHRENHEIT, CELSIUS }

Thermometer.newInstance（TemperatureScale.CELSIUS）不仅比Thermometer.newInstance（true）更有意义，而且您可以在将来的版本中将KELVIN添加到TemperatureScale，而无需向Thermometer添加新的静态工厂。 此外，您可以将温度计依赖关系重构为枚举常量的方法（第34项）。 例如，每个缩放常量可以有一个采用double值并将其转换为Celsius的方法。
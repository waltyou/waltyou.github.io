---
layout: post
title: 《Effective Java》学习日志（六）43：方法引用优于lambda表达式
date: 2018-11-27 18:40:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

lambda优于匿名类的主要优点是它更简洁。Java提供了一种生成函数对象的方法，比lambda还要简洁，那就是：方法引用（ method references）。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

下面是一段程序代码片段，它维护一个从任意键到整数值的映射。
如果将该值解释为键的实例个数，则该程序是一个多重集合的实现。
该代码的功能是，根据键找到整数值，然后在此基础上加1：

    map.merge(key, 1, (count, incr) -> count + incr);

请注意，此代码使用merge方法，该方法已添加到Java 8中的Map接口中。
如果没有给定键的映射，则该方法只是插入给定值; 如果映射已经存在，则合并给定函数应用于当前值和给定值，并用结果覆盖当前值。 
此代码表示merge方法的典型用例。

代码很好读，但仍然有一些样板的味道。 
参数count和incr不会增加太多价值，并且占用相当大的空间。 
真的，所有的lambda都告诉你函数返回两个参数的和。 
从Java 8开始，Integer类（和所有其他包装数字基本类型）提供了一个静态方法总和，和它完全相同。 
我们可以简单地传递一个对这个方法的引用，并以较少的视觉混乱得到相同的结果：

    map.merge(key, 1, Integer::sum);

方法的参数越多，你可以通过方法引用消除更多的样板。 
然而，在一些lambda中，选择的参数名称提供了有用的文档，使得lambda比方法引用更具可读性和可维护性，即使lambda看起来更长。

对于一个方法引用，你无能为力，因为你不能对lambda执行任何操作（只有一个难懂的异常 - 如果你好奇的话，参见JLS，9.9-2）。 
也就是说，方法引用通常会导致更短，更清晰的代码。 
如果lambda变得太长或太复杂，它们也会给你一个结果：
你可以从lambda中提取代码到一个新的方法中，并用对该方法的引用代替lambda。 
你可以给这个方法一个好名字，并把它文档记录下来。

如果你使用IDE编程，它将提供替换lambda的方法，并在任何地方使用方法引用。
通常情况下，你应该接受这个提议。

偶尔，lambda会比方法引用更简洁。
这种情况经常发生在方法与lambda相同的类中。

例如，考虑这段代码，它被假定出现在一个名为GoshThisClassNameIsHumongous的类中：

    service.execute(GoshThisClassNameIsHumongous::action);

这个lambda类似于等价于下面的代码：

    service.execute(() -> action());

使用方法引用的代码段既不比使用lambda的代码片段更短也不清晰，所以更喜欢后者。 
在类似的代码行中，Function接口提供了一个通用的静态工厂方法来返回标识函数Function.identity()。 
它通常更短，更清洁，而不使用这种方法，而是使用等效的lambda内联代码：x - > x。

许多方法引用是指静态方法，但有四种方法没有。 
其中两个是特定（bound）和任意（unbound）对象方法引用。 
在特定对象引用中，接收对象在方法引用中指定。 
特定对象引用在本质上与静态引用类似：函数对象与引用的方法具有相同的参数。 
在任意对象引用中，接收对象在应用函数对象时通过方法的声明参数之前的附加参数指定。 
任意对象引用通常用作流管道（pipelines）中的映射和过滤方法（条目 45）。 

最后，对于类和数组，有两种构造方法引用。 构造方法引用用作工厂对象。 
下表总结了所有五种方法引用：

[![](/images/posts/effective-java-42.jpg)](/images/posts/effective-java-42.jpg)

总之，方法引用通常为lambda提供一个更简洁的选择。 如果方法引用看起来更简短更清晰，请使用它们；否则，还是坚持lambda。

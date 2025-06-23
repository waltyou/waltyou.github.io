---
layout: post
title: 《Effective Java》学习日志（三）21：为后代设计接口
date: 2018-10-30 10:51:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

在 Java 8 之前，在不打破实现的情况下，很难给接口添加新的方法。 
但是增加到新方法到现有接口的行为，还是充满风险的。

<!-- more -->
---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---



* 目录
{:toc}

---

默认方法包含了一个默认实现，这个默认实现可以在所有实现了该接口的类里使用，除非这个类自己重写了这个默认方法。
所以当给接口添加一个默认方法的时候，我们无法保证这个方法，能在所有实现这个接口的类中生效。
默认的方法被“注入（injected）”到现有的实现中，没有经过实现类的知道或同意。 
在Java 8之前，这些实现是用默认的接口编写的，它们的接口永远不会获得任何新的方法。

许多新的默认方法被添加到Java 8的核心集合接口中，主要是为了方便使用lambda表达式（第6章）。 
Java类库的默认方法是高质量的通用实现，在大多数情况下，它们工作正常。 
但是，要写一个能够维护每一个实现中的所有变量的默认方法，是不太可能的。

例如，考虑在Java 8中添加到Collection接口的removeIf方法。
此方法删除给定布尔方法（或Predicate函数式接口）返回true的所有元素。
默认实现被指定为使用迭代器遍历集合，调用每个元素的谓词，并使用迭代器的remove方法删除谓词返回true的元素。 
据推测，这个声明看起来像这样：默认实现被指定为使用迭代器遍历集合，调用每个元素的Predicate函数式接口，并使用迭代器的remove方法删除Predicate函数式接口返回true的元素。 

根据推测，这个声明看起来像这样：

```java
// Default method added to the Collection interface in Java 8
default boolean removeIf(Predicate<? super E> filter) {
    Objects.requireNonNull(filter);
    boolean result = false;
    for (Iterator<E> it = iterator(); it.hasNext(); ) {
        if (filter.test(it.next())) {
            it.remove();
            result = true;
        }
    }
    return result;
}
```

这是可能为removeIf方法编写的最好的通用实现，但遗憾的是，它在一些实际的Collection实现中失败了。 

例如，org.apache.commons.collections4.collection.SynchronizedCollection 方法。 
这个类出自Apache Commons类库中，与java.util包中的静态工厂Collections.synchronizedCollection方法返回的类相似。 
Apache版本还提供了使用客户端提供的对象进行锁定的能力，以代替集合。 
换句话说，它是一个包装类（条目 18），它们的所有方法在委托给包装集合类之前在一个锁定对象上进行同步。

Apache的SynchronizedCollection类仍然在积极维护，但在撰写本文时，并未重写removeIf方法。 
如果这个类与Java 8一起使用，它将继承removeIf的默认实现，但实际上不能保持类的基本承诺：自动同步每个方法调用。 
默认实现对同步一无所知，并且不能访问包含锁定对象的属性。 
如果客户端在另一个线程同时修改集合的情况下调用SynchronizedCollection实例上的removeIf方法，则可能会导致ConcurrentModificationException异常或其他未指定的行为。

为了防止在类似的Java平台类库实现中发生这种情况，比如Collections.synchronizedCollection返回的包级私有的类，JDK维护者必须重写默认的removeIf实现和其他类似的方法来在调用默认实现之前执行必要的同步。 
原来不属于Java平台的集合实现没有机会与接口更改进行类似的改变，有些还没有这样做。

在默认方法的情况下，接口的现有实现类可以在没有错误或警告的情况下编译，但在运行时会失败。 
虽然不是非常普遍，但这个问题也不是一个孤立的事件。 
在Java 8中添加到集合接口的一些方法已知是易受影响的，并且已知一些现有的实现会受到影响。

应该避免使用默认方法向现有的接口添加新的方法，除非这个需要是关键的，在这种情况下，你应该仔细考虑，以确定现有的接口实现是否会被默认的方法实现所破坏。
然而，默认方法对于在创建接口时提供标准的方法实现非常有用，以减轻实现接口的任务（条目 20）。

还值得注意的是，默认方法不是被用来设计，来支持从接口中移除方法或者改变现有方法的签名的目的。
在不破坏现有客户端的情况下，这些接口都不可能发生更改。

准则是清楚的。 
尽管默认方法现在是Java平台的一部分，但是非常悉心地设计接口仍然是非常重要的。 
虽然默认方法可以将方法添加到现有的接口，但这样做有很大的风险。 
如果一个接口包含一个小缺陷，可能会永远惹怒用户。 
如果一个接口严重缺陷，可能会破坏包含它的API。

因此，在发布之前测试每个新接口是非常重要的。 
多个程序员应该以不同的方式实现每个接口。 
至少，你应该准备三种不同的实现。 
编写多个使用每个新接口的实例来执行各种任务的客户端程序同样重要。 
这将大大确保每个接口都能满足其所有的预期用途。 
这些步骤将允许你在发布之前发现接口中的缺陷，但仍然可以轻松地修正它们。 
虽然在接口被发布后可能会修正一些存在的缺陷，但不要太指望这一点。

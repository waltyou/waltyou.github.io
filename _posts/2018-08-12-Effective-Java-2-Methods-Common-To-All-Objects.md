---
layout: post
title: 《Effective Java》学习日志（二）：Object的通用方法
date: 2018-08-12 21:11:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

这一篇主要谈一下 Object 类中的通用方法。

<!-- more -->
---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---
## 目录
{:.no_toc}

* 目录
{:toc}

---

# Item 10：重写 equals 方法时遵守的通用约定

虽然Object是一个具体的类，但它主要是为继承而设计的。它的所有非 final方法(equals、hashCode、toString、clone和finalize)都有清晰的通用约定（ general contracts），因为它们被设计为被子类重写。任何类都有义务重写这些方法，以遵从他们的通用约定；如果不这样做，将会阻止其他依赖于约定的类(例如HashMap和HashSet)与此类一起正常工作。

重写equals方法看起来很简单，但是有很多方式会导致重写出错，同时其结果可能是可怕的。避免此问题最简单的方法是不覆盖equals方法，但是在这种情况下，类的每个实例只与自身相等。

如果满足以下任一下条件，则说明是正确的做法：
- **每个类的实例都是固有唯一的**。对于像Thread这样代表活动实体而不是值的类来说，这是正确的。 Object提供的equals实现对这些类完全是正确的行为。
- **类不需要提供一个“逻辑相等（logical equality）”的测试功能**。例如java.util.regex.Pattern可以重写equals 方法检查两个是否代表完全相同的正则表达式Pattern实例，但是设计者并不认为客户需要或希望使用此功能。在这种情况下，从Object继承的equals实现是最合适的。
- **父类已经重写了equals方法，则父类行为完全适合于该子类**。例如，大多数Set从AbstractSet继承了equals实现、List从AbstractList继承了equals实现，Map从AbstractMap的Map继承了equals实现。
- **类是私有的或包级私有的，可以确定它的equals方法永远不会被调用**。如果你非常厌恶风险，可以重写equals方法，以确保不会被意外调用：
    ```java
    @Override public boolean equals(Object o) {
        throw new AssertionError(); // Method is never called
    }
    ```

那什么时候需要重写 equals 方法呢？如果一个类包含一个逻辑相等（ logical equality）的概念，此概念有别于对象标识（object identity），而且父类还没有重写过equals 方法。这通常用在值类（ value classes）的情况。值类只是一个表示值的类，例如Integer或String类。程序员使用equals方法比较值对象的引用，期望发现它们在逻辑上是否相等，而不是引用相同的对象。重写 equals方法不仅可以满足程序员的期望，它还支持重写过equals 的实例作为Map 的键（key），或者 Set 里的元素，以满足预期和期望的行为。

一种不需要equals方法重写的值类是使用实例控制（instance control）（Item 1）的类，以确保每个值至多存在一个对象。 枚举类型（Itme 34）属于这个类别。对于这些类，逻辑相等与对象标识是一样的，所以Object的equals方法作用逻辑equals方法。

当你重写equals方法时，必须遵守它的通用约定。Object的规范如下：equals方法实现了一个等价关系（equivalence relation）。它有以下这些属性:
- **自反性**：对于任何非空引用x，x.equals(x)必须返回true。
- **对称性**：对于任何非空引用x和y，如果且仅当y.equals(x)返回true时x.equals(y)必须返回true。
- **传递性**：对于任何非空引用x、y、z，如果x.equals(y)返回true，y.equals(z)返回true，则x.equals(z)必须返回true。
- **一致性**：对于任何非空引用x和y，如果在equals比较中使用的信息没有修改，则x.equals(y)的多次调用必须始终返回true或始终返回false。
- 对于任何非空引用x，x.equals(null)必须返回false。

如果一旦违反了它，很可能会发现你的程序运行异常或崩溃，并且很难确定失败的根源。套用约翰·多恩(John Donne)的说法，没有哪个类是孤立存在的。一个类的实例常常被传递给另一个类的实例。许多类，包括所有的集合类，都依赖于传递给它们遵守equals约定的对象。


---

# 未完待续.....

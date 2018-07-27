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

## 优点

### 1）静态工厂方法有名字

当构造函数的参数本身不能很好的描述函数返回的是什么样的对象时，一个有好名字的静态方法，会帮助客户端代码更好的理解。

举个例子就是：构造函数 BigInteger(int, int, Random)，它返回了一个可能是素数的 BigIntege， 但是如果使用静态工厂方法 BigInteger.probablePrime ，表达就会更加清晰。

另外，我们都知道对于给定的一个标识，一个类只能有一个对应的构造函数。但有时候，为了打破这个限制，程序员可能会使用两个仅仅参数顺序不一致的构造函数来解决这个问题。这是个很不好的行为。因为使用者很可能分不清哪个构造函数该被使用，从而导致错误发生。除非他们认真的阅读使用文档。

但是静态工厂方法的名字就解决了上述问题，只需要取两个定义清晰且不同的名字就可以了。
 
### 2）静态工厂方法不需要每次都创建新对象

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

### 3）静态工厂方法可以返回其返回类型的任何子类型对象

这个功能让使用者可以更加灵活地选择返回对象的类。

这个灵活性的一个应用就是 API 可以在不使返回对象类公开的情况下，返回一个对象。只需要返回对象的类，是静态工厂方法定义时规定的返回类型的子类即可。这项技术适用于基于接口的框架（interface-based frameworks），这里的接口提供对象的原生返回类型。

按照惯例，一个名字是“Type”的接口，它的静态工厂方法，通常都会放在一个名为“Types”的不可实例化的伴随类中。例如，Java Collections Framework的接口有45个实用程序实现，提供不可修改的集合，同步集合等。几乎所有这些实现都是通过静态工厂方法在一个不可实例化的类（java.util.Collections）中导出的。返回对象的类都是非公共的。

借助这种技术，Collections 类就变的小了很多。这不仅仅是API大部分的减少，也包括概念上的重量：程序员为使用API必须掌握概念的数量和难度。程序员知道返回的对象具有其接口指定的API，因此不需要为这个实现类而阅读额外的类文档。

此外，使用这种静态工厂方法，需要客户端通过**接口**而不是**实现类**来引用返回的对象，这通常是一种很好的做法。

从Java 8开始，消除了接口不能包含静态方法的限制，因此通常没有理由为接口提供不可实例化的伴随类。许多公共静态成员应该放在接口本身中。但请注意，可能仍有必要将大量实现代码放在这些静态方法后面的单独的包私有（package-private）类中。这是因为Java 8要求接口的所有静态成员都是公共的。Java 9允许私有静态方法，但静态字段和静态成员类仍然需要公开。

### 4）静态工厂方法可以根据输入参数而改变返回对象的类

返回对象的类型，只要是声明类型的子类型就可以。

EnumSet 类就没有公共的构造方法，只有静态工厂。在 OpenJDk 的实现上，它可以返回两个子类型中的其中一种：如果 enum type 数量小于等于64，静态工厂会返回 RegularEnumSet，否则，会返回 JumboEnumSet 。

这两种实现的子类，对于调用者是不可见的。所以，如果将来出于性能考虑，移除这个类，那对使用者也毫无影响。同样的，再添加一个新的子类，对调用者也无影响。

### 5）在写静态工厂方法时，方法返回对象的类不需要存在。

这种灵活的静态工厂方法构成了服务提供者框架（service provider frameworks）的基础，如Java数据库连接API（JDBC）。服务提供者框架是提供者负责实现服务的系统。系统使实现可用于客户端，将客户端与实现分离。

服务提供者框架中有三个基本组件：
- service interface，代表一个具体实现
- provider registration API，提供者用于注册一个实现
- service access API，客户端使用它来获取服务的实例

Service access API可以允许客户端指定用于选择实现的标准，如果没有这样的标准，API将返回默认实现的实例，或允许客户端循环遍历所有可用的实现。 Service access API是灵活的静态工厂，它构成了服务提供者框架的基础。

另外一个可选的组件是：service provider interface，它用来描述一个生产service interface实例的工厂对象。在缺少服务提供者接口的情况下，必须反射地实例化实现。在 JDBC 中, Connection 作为 service interface, DriverManager.registerDriver 作为 provider registration API, DriverManager.getConnection 作为 service access API, Driver 是 service provider interface.

服务提供者框架模式有许多变体。 例如，服务访问API可以向客户端返回比提供者提供的服务接口更丰富的服务接口。这就是桥接模式(Bridge Pattern)。 依赖注入框架也看做是强大的服务提供者。 Java 6 提供了通用目的的服务提供者框架：java.util.ServiceLoader，所以你无需自己实现。

## 局限性

### 1）没有 public 或 protected 构造函数的类不能被子类化

例如，我们不可能在Collections Framework中继承任何便捷的实现类。

可以说这可能是一种伪装的祝福，因为它鼓励程序员使用组合而不是继承（第18项），并且是不可变类型（第17项）所必需的。

### 2）静态工厂方法不能容易的被使用者找到

构造方法，我们不看 API 文档也知道，但是静态工厂方法不一样，所以我们最好约定一些命名规范，来减少问题的发生。如下：

- from: 一种类型转换方法，它接受一个参数并返回一个相应的这种类型的实例。
    ```
    Date d = Date.from(instant);
    ```
- of：一种聚合方法，它接受多个参数并返回实例包含它们的这种类型
    ```
    Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING);
    ```
- valueOf：一个更详细的替代 from 和 of
    ```
    BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE);
    ```
- instance or getInstance：返回由其参数（如果有）描述的实例，但不能说具有相同的值
    ```
    StackWalker luke = StackWalker.getInstance(options);
    ```
- create or newInstance：像instance或getInstance，但是该方法保证每个调用返回一个新实例
    ```
    Object newArray = Array.newInstance(classObject, arrayLen);
    ```
- getType：与getInstance类似，但如果工厂方法位于不同的类中，则使用它。 Type是工厂方法返回的对象类型
    ```
    FileStore fs = Files.getFileStore(path);
    ```
- newType：与newInstance类似，但如果工厂方法在不同的类中，则使用。 Type是工厂方法返回的对象类型
    ```
    BufferedReader br = Files.newBufferedReader(path);
    ```
- type：getType和newType的简明替代方案
    ```
    List<Complaint> litany = Collections.list(legacyLitany);
    ```

---

# Item 2：当构造函数有许多参数的时，请考虑构建器（Builder）

静态工厂和构造函数共享一个限制：当有很多可选参，它们不能很好地扩展。

因为面对这种可选参数较多的情况，构造函数无论如何都需要传递一个值给它，即使这些参数我们不需要。

直观上，我们可以采用**伸缩构造模式**的方法（也就是函数复用），来一定程度上解决这个问题。但是当参数变得更多时，这个思路下代码就会臃肿起来。而且程序也变得更加难以阅读。

第二个思路是**JavaBeans**模式，也就是使用 get、set 方法。您可以在其中调用无参数构造函数来创建对象，然后调用setter方法来设置每个必需参数和每个感兴趣的可选参数。这个方法没有上一个方法的缺点。它很容易创建实例，并且易于阅读生成的代码。

不幸的是，JavaBeans模式本身就存在严重的缺点。因为想要构造出一个完整地对象，需要多次调用，而这些调用在多线程的情况下，可以会出现不一致的状态。当然我们可以使用锁来避免这类错误，但是程序就变得笨重了。

幸运的是，这里有第三种方式，就是生成器模式（Builder Pattern）。

---

# 未完待续.....

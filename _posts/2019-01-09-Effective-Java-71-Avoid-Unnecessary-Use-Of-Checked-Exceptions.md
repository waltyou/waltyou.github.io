---
layout: post
title: 《Effective Java》学习日志（九）71:避免使用无必要的受检异常
date: 2019-01-09 19:59:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]


---

<!-- more -->

------

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

------

## 目录
{:.no_toc}

* 目录
{:toc}

------

许多Java程序员不喜欢检查异常，但如果使用得当，他们可以改进API和程序。与返回代码和未经检查的异常不同，它们迫使程序员处理问题，增强可靠性。也就是说，在API中过度使用已检查的异常会使它们使用起来不那么令人愉快。如果方法抛出已检查的异常，则调用它的代码必须在一个或多个**catch**块中处理它们，或者声明它抛出它们并让它们向外传播。无论哪种方式，它都会给API的用户带来负担。 Java 8中的负担增加，因为抛出已检查异常的方法不能直接在流中使用（项目45-48）。

如果通过正确使用API无法防止异常情况，并且使用API的程序员在遇到异常时可以采取一些有用的操作，则这种负担可能是合理的。除非满足这两个条件，否则未经检查的异常是合适的。作为试金石，请问自己程序员将如何处理异常。这是最好的吗？

```java
} catch (TheCheckedException e) {
	throw new AssertionError(); // Can't happen!
}
```

或者这个？

```java
} catch (TheCheckedException e) {
    e.printStackTrace(); // Oh well, we lose.
    System.exit(1);
}
```

如果程序员不能做得更好，则需要调用未经检查的异常。

如果它是由方法抛出的唯一检查异常，则由检查异常引起的程序员的额外负担要大得多。如果还有其他方法，则该方法必须已出现在try块中，并且此异常最多需要另一个catch块。如果方法抛出单个已检查的异常，则此异常是该方法必须出现在try块中且不能直接在流中使用的唯一原因。在这种情况下，问问自己是否有办法避免检查异常是值得的。

消除已检查异常的最简单方法是返回所需结果类型的可选项（第55项）。该方法只返回一个空的可选项，而不是抛出一个已检查的异常。该技术的缺点在于该方法不能返回任何附加信息，详细说明其无法执行所需的计算。相反，例外情况具有描述性类型，可以导出方法以提供额外信息（项目70）。

您还可以通过将抛出异常的方法分解为两个方法来将已检查的异常转换为未经检查的异常，第一个方法返回一个布尔值，指示是否抛出异常。此API重构会从以下内容转换调用序列：

```java
// Invocation with checked exception
try {
	obj.action(args);
} catch (TheCheckedException e) {
	... // Handle exceptional condition
}
```

成为这样：

```java
// Invocation with state-testing method and unchecked exception
if (obj.actionPermitted(args)) {
	obj.action(args);
} else {
	... // Handle exceptional condition
}
```

这种重构并不总是合适的，但它可以使API更加舒适。 虽然后一个调用序列并不比前者更漂亮，但重构的API更灵活。 如果程序员知道调用会成功，或者满足于让线程在失败时终止，那么重构也允许这个简单的调用序列：

```java
obj.action(args);
```

如果您怀疑普通的调用序列将是常态，那么API重构可能是合适的。生成的API本质上是第69项中的状态测试方法API，并且适用相同的注意事项：如果要在没有外部同步的情况下同时访问对象，或者它受到外部诱导的状态转换，则此重构是不合适的，因为对象的状态可能是在对**actionPermitted**和**action**的调用之间进行更改。如果单独的**actionPermitted**方法会复制**action**方法的工作，则可能会因性能原因而排除重构。

总之，在谨慎使用时，检查异常可以提高程序的可靠性;当过度使用时，它们会使API难以使用。如果调用者无法从失败中恢复，则抛出未经检查的异常。如果可能进行恢复并且您希望强制调用者处理异常情况，请首先考虑返回可选项。只有在失败的情况下提供的信息不足时才应该抛出一个已检查的异常。








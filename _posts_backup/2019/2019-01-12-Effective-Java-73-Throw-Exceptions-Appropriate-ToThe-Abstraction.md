---
layout: post
title: 《Effective Java》学习日志（九）73:抛出适合抽象的异常
date: 2019-01-12 18:20:04
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

当一个方法抛出一个与它执行的任务没有明显联系的异常时，这是令人不安的。 当一个方法传播由低级抽象抛出的异常时，通常会发生这种情况。 它不仅令人不安，而且还通过实现细节污染了更高层的API。 如果更高层的实现在以后的版本中发生更改，则它抛出的异常也会发生更改，从而可能会破坏现有的客户端程序。

为了避免这个问题，**更高层应该捕获较低级别的异常，并且在它们的位置抛出可以用更高级别的抽象来解释的异常**。 这个惯用语法被称为异常翻译（exception translation）：

```
// Exception Translation
try {
	... // Use lower-level abstraction to do our bidding
} catch (LowerLevelException e) {
	throw new HigherLevelException(...);
}
```

以下是从AbstractSequentialList类获取的异常转换示例，该类是List接口的骨干实现（第20项）。 在此示例中，异常转换由List <E>接口中的get方法规范强制要求：

```java
/**
* Returns the element at the specified position in this list.
* @throws IndexOutOfBoundsException if the index is out of range
*
({@code index < 0 || index >= size()}).
*/
public E get(int index) {
    ListIterator<E> i = listIterator(index);
    try {
    	return i.next();
    } catch (NoSuchElementException e) {
    	throw new IndexOutOfBoundsException("Index: " + index);
    }
}
```

如果较低级别的异常可能对调试导致更高级别异常的问题的某人有帮助，则需要一种称为异常链接的特殊形式的异常链接。 较低级别的异常（原因）被传递给更高级别的异常，它提供了一个访问器方法（Throwable的getCause方法）来检索较低级别的异常：

```java
// Exception Chaining
try {
	... // Use lower-level abstraction to do our bidding
} catch (LowerLevelException cause) {
	throw new HigherLevelException(cause);
}
```

高级异常的构造函数将原因传递给链接感知超类构造函数，因此它最终传递给Throwable的一个链接感知构造函数，例如Throwable（Throwable）：

```java
// Exception with chaining-aware constructor
class HigherLevelException extends Exception {
    HigherLevelException(Throwable cause) {
    	super(cause);
    }
}
```

大多数标准异常都具有链接感知构造函数。对于没有的异常，可以使用Throwable的initCause方法设置原因。异常链接不仅允许您以编程方式访问原因（使用getCause），而且它还将原因的堆栈跟踪集成到更高级别异常的跟踪中。

**虽然异常转换优于低级别异常的无意识传播，但不应过度使用。**在可能的情况下，处理较低层异常的最佳方法是通过确保较低级别的方法成功来避免它们。有时您可以通过检查更高级别方法的参数的有效性，然后再将它们传递到较低层来完成此操作。

如果不可能防止来自较低层的异常，那么下一个最好的事情是让较高层静默地解决这些异常，从而使较高级别方法的调用者与较低级别的问题隔离开来。在这些情况下，使用某些适当的日志记录工具（如java.util.logging）记录异常可能是适当的。这允许程序员调查问题，同时使客户端代码和用户与其隔离。

总之，如果不可能阻止或处理较低层的异常，请使用异常转换，除非较低级别的方法恰好保证其所有异常都适用于较高级别。链接提供了两全其美的优点：它允许您抛出适当的更高级别异常，同时捕获失败分析的根本原因（第75项）。


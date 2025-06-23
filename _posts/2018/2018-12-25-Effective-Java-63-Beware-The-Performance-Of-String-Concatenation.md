---
layout: post
title: 《Effective Java》学习日志（八）63:注意string合并的性能
date: 2018-12-25 18:50:04
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

字符串连接运算符（+）是将几个字符串合并为一个的便捷方式。 它可以生成单行输出或构造一个小的固定大小对象的字符串表示，但它不能缩放。 **重复使用 + 进行字符串拼接的时间复杂度是O(N)**。 这是字符串不可变这一事实的不幸后果（第17项）。 当连接两个字符串时，将复制两者的内容。

例如，考虑这种方法，它通过重复连接每个项目的一行来构造一个记帐语句的字符串表示：

```java
// Inappropriate use of string concatenation - Performs poorly!
public String statement() {
    String result = "";
    for (int i = 0; i < numItems(); i++)
    	result += lineForItem(i); // String concatenation
    return result;
}
```

如果项目数量很大，则该方法执行得非常糟糕。 **要获得可接受的性能，请使用StringBuilder代替String**来存储正在构造的语句：

```java
public String statement() {
    StringBuilder b = new StringBuilder(numItems() * LINE_WIDTH);
    for (int i = 0; i < numItems(); i++)
    	b.append(lineForItem(i));
    return b.toString();
}
```

自Java 6以来，许多工作已经开始使字符串连接更快，但两种方法的性能差异仍然很大：如果numItems返回100而lineForItem返回80个字符的字符串，则第二个方法的运行速度比第一个快6.5倍。 因为第一种方法的项目数量是二次方的，而第二种方法是线性的，所以随着项目数量的增长，性能差异会变得更大。 请注意，第二种方法预先分配一个足够大的StringBuilder来保存整个结果，从而无需自动增长。 即使使用默认大小的StringBuilder失谐，它仍然比第一种方法快5.5倍。

基本原则很简单：**除非性能无关紧要，否则不要使用字符串连接运算符来组合多个字符串**。 请改用StringBuilder的append方法。 或者，使用字符数组，或一次处理一个字符串而不是组合它们。
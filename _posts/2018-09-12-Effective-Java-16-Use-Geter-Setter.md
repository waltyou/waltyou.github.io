---
layout: post
title: 《Effective Java》学习日志（三）16：使用存取器方法（setter or getter）代替public属性
date: 2018-09-12 09:41:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

使用 setter or getter 有诸多好处，来一起看看。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

有时候你可能会写一些很简单的类来把一些实例属性聚集在一起，类似于下面的类：

```java
//这种退化的类不应该被设置成public
class Point {
	public double x;
	public double y;
}
```

这种类的属性可以被直接访问，所以会破坏封装性（Item 15）。
因为当你想调整这个类就必须同时调整相应的API，也不能强制使这些变量变得不可变，而且当变量被改变时你也很难做一些特殊的额外操作。
这样做是有悖于面向对象开发的，在面向对象开发中一般会把它改成私有的变量，然后设置他们的存取器方法（getter和setter）：

```java
class Point {
	private double x;
	private double y;
	public Point(double x, double y) {
		this.x = x;
		this.y = y;
	}
	public double getX() { return x; }
	public double getY() { return y; }
	public void setX(double x) { this.x = x; }
	public void setY(double y) { this.y = y; }
}
```

当一个类是公共类时应该强制遵守这样的规则：如果一个类可以在它的包外部被访问到，就应该提供访问器方法来保持类可被修改的灵活性。
如果你的类公开了自己数据域，那么将来当你想修改的时候就要考虑已经基于你的这些数据域开发的客户端可能已经遍布全球，你也应该有责任不影响它们的使用，
所以此时你想修改这些数据域将会面临困难。

然而，如果一个类是 package-private 或者是一个私有的内部嵌套类，
那么公开它的数据域本身就没有什么错误—假设是这些数据域已经足以描述类的抽象性。

这种方式无论是在定义的时候还是在被其他人调用的时候都比访问器看起来更加简洁。

你可能会想这些类的内部数据不也会被调用者绑定并修改么？但此时也只有包内部的类才有权限修改。
而且如果是内部类，修改的范围会更小一步被限制在包含内部类的类中。

Java库中的很多类也违反了这条规则，直接公开了一些属性。
最常见的例子就是在java.awt包中的Point 和 Dimension类。
这样的类我们应该摒弃而不是模仿。

在 Item 67 中会介绍到，因为Dimension类的内部实现对外暴露，造成一系列性能问题一直延续到今天都没能解决。

虽然公共的类直接暴露属性给外部不可取，但是如果暴露的是不可变对象，危险就会降低很多。
因为这时候你不能修改类的内部数据，也不能当对象被读取的时候做一些违规操作，你能做的是更加加强其不可变性。
例如下面例子中的每一个实例变量代表一种时间：

```java
//公共类暴露不可变对象
public final class Time {
	private static final int HOURS_PER_DAY = 24;
	private static final int MINUTES_PER_HOUR = 60;
    
	public final int hour;
	public final int minute;
    
	public Time(int hour, int minute) {
		if (hour < 0 || hour >= HOURS_PER_DAY)
			throw new IllegalArgumentException("Hour: " + hour);
		if (minute < 0 || minute >= MINUTES_PER_HOUR)
			throw new IllegalArgumentException("Min: " + minute);
		this.hour = hour;
		this.minute = minute;
	}
	... // Remainder omitted
}
```

总结起来就是，public的类不能暴露可变对象给外部。
即使是不可变对象也同样具有一定危害。
然而，有时在一些设计中 package-private 或嵌套内部类中还是需要暴露可变或不可变对象的。


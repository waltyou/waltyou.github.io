---
layout: post
title: Java 并发编程实战-学习笔记（一）1：线程安全
date: 2019-05-29 15:18:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

进入第一部分的第一章节：并发基础之线程安全。

<!-- more -->

* 目录
{:toc}
---

# 什么是线程安全

书中定义：

> 当多个线程访问某个类时，不管运行时环境采用何种调度方式或者这些线程将如何交替执行，并且在主调代码中不需要任何额外的同步或协同，这个类就能表现出正确的行为，那么就称这个类是线程安全的。

自己理解：

> 一个类或者方法在多线程情况下，不管怎么使用它，它的结果都符合预期，这就是线程安全。



无状态的对象，天生是线程安全，因为无状态对象不包含任何域，也不包含任何对其他域的引用。



# 原子性

关于原子性的一个例子就是多线程下，对一个变量进行加一操作：

```java
count ++；
```

虽然看似是一行代码，但是其实包含了三个动作：读出来，加一，写回去。所以在多线程情况下，可能会出现两个线程都读了原始值，比如是1，然后各自加一，再写回去。这时候我们预期结果是3，但是实际结果却为 2。

这种由于执行时序而出现不正确的情况，学名叫做：竞态条件。

## 1. 竞态条件

当某个计算的正确性取决于多个线程的交替执行时序时，就会发生竞态条件。

最常见的竞态条件类型就是“先检查后执行（Check-then-Act）”，即通过一个可能失效的观测结果来决定下一步的动作。

一个常见的例子就是单例模式中单例的初始化。

```java
@NotThreadSafe
public class LazyInitRace{
	private ExpensiveObject instance = null;
	
	public ExpensiveObject getInstance(){
		if(instance == null){
			instance = new ExpensiveObject();
		}
		return instance;
	}
}
```

以上代码，就存在一个竞态条件，因为有可能会有两个线程同时进行 instance 是否为 null 判断的情况，那么这两个线程都可能创建一个新的 instance  的。

## 2. 复合操作

所谓复合操作就是指包含了一组以原子方式执行的操作。

# 加锁机制

## 1. 内置锁

关键字： **synchronized** 。

同步代码块，包含两个部分：1，作为锁的对象引用，2，作为这个锁保护的代码块。

以 synchronized 来修饰的方法，它持有的锁就是方法调用所在的对象；静态的 synchronized 方法的锁，是 Class 对象。
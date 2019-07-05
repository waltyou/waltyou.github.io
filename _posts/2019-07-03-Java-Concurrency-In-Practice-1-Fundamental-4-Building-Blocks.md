---
layout: post
title: Java 并发编程实战-学习日志（一）4：基础构建模块
date: 2019-07-03 21:13:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

这一章主要介绍 Java 中有用的并发模块。

<!-- more -->

* 目录
{:toc}
---

# 同步容器类

常见的有 Vector 和 Hashtable，还有一些通过 Collections.synchronizedXxx 等工厂方法创建的同步容器，它们实现同步的方式都是：将状态封装起来，并对每个公有方法都进行同步，使得每次只有一个线程能访问容器的状态。

## 1. 同步容器类的问题

同步容器都是线程安全的，但在某些情况下可能需要额外的客户端加锁操作来保护符合操作。在容器上常见的符合操作包括：迭代、跳转以及“先检查再执行”等。

### 先检查再执行

```java
public static Object getLast(Vector list){
    int lastIndex = list.size()-1;
    return list.get(lastIndex);
}

public static Object deleteLast(Vector list){
    int lastIndex = list.size()-1;
    list.remove(lastIndex);
}
```

### 迭代

```java
for (int i = 0; i < vector.size(); i++)
  doSomething(vector.get(i));
```

上面的代码在多线程情况下，可能会抛出 ArrayIndexOutOfBoundsException。



## 2. 迭代器与 ConcurrentModificationException

对容器进行迭代的标准方式是使用迭代器Iterator。然而在设计同步容器的迭代器时并没有考虑并发修改问题，如果在对同步容器进行迭代的过程中有线程修改了容器，那么就会失败。并且它们的表现行为是“及时失败”的。这意味着当它们发现容器在迭代过程中被修改时，就会抛出ConcurrentMondificationException异常。

```java
List<Widget> widgetList = Collections.synchronizedList(new ArrayList<Widget>());
...
// May throw ConcurrentModificationException
for (Widget w : widgetList)
  doSomething(w);
```

## 3. 隐藏迭代器

```Java
public class HiddenIterator {
  @GuardedBy("this")
  private final Set<Integer> set = new HashSet<Integer>();
  public synchronized void add(Integer i) { set.add(i); } 
  public synchronized void remove(Integer i) { set.remove(i); }
  
  public void addTenThings() { 
    Random r = new Random();
    for (int i = 0; i < 10; i++)
			add(r.nextInt());
		System.out.println("DEBUG: added ten elements to " + set);
  }
}
```

上面代码在 System.out.println 时就可能产生异常，因为 set 的 toString 方法会对set 进行迭代。

> 正如封装对象的状态有助于维持不变性条件一样，封装对象的同步机制同样有助于确保实施同步策略。

### 未完待续。。。。。





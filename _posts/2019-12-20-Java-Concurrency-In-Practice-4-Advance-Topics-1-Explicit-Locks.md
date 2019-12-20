---
layout: post
title: Java 并发编程实战-学习日志（四）1：显式锁
date: 2019-12-20 17:11:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

在Java5.0之前，在协调对共享对象的访问时可以使用的机制只有synchronized和volatile。Java5.0增加了一种新的机制：ReentrantLock。与之前提到过的机制相反，ReentrantLock并不是一种替代内置加锁的方法，而是当内置加锁机制不适应时，作为一种可选的高级功能。

<!-- more -->

---

* 目录
{:toc}
---

## Lock与ReentrantLock

与内置加锁机制不同的是，Lock提供了一种无条件的、可轮询的、定时的以及可中断的锁获取操作，所有加锁和解锁的方法都是显式的。在Lock的实现中必须提供与内部锁相同的内存可见性语义，但在加锁语义、调度算法、顺序保证以及性能特性等方面可以有所不同。

```
public interface Lock {
  void lock();
  void lockInterruptibly() throws InterruptedException;
  boolean tryLock();
  boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
  void unlock();
  Condition newCondition();
}
```

ReentrantLock实现了Lock接口，并提供了与synchronized相同的排斥性和内存可见性。在获取ReentrantLock时，有着与进入同步代码块相同的内存语义，在释放ReentrantLock时，同样有着与退出同步代码块相同的内存语义。此外，与synchronized一样，ReentrantLock还提供了可重入的加锁语义。ReentrantLock支持在Lock接口中定义的多有获取锁模式，并且与synchronized相比，它还为处理锁的不可用性问题提供了更高的灵活性。

为什么要创建一种与内置锁如此相似的新加锁机制？在大多数情况下，内置锁都能很好地工作，但在功能上存在一些局限性，例如，无法中断一个正在等待获取锁的线程，或者无法在请求获取一个锁时无限地等待下去。内置锁必须在获取该锁的代码块中释放，这就简化了编码工作，并且与异常处理操作实现了很好的交互，但却无法实现非阻塞结构的加锁规则。这些都是使用synchronized的原因，但在某些情况下，一种更灵活的加锁机制通常能提供更好的活跃性或性能。

Lock锁的使用比使用内置锁复杂一些：必须在finally块中释放锁。否则，如果在被保护的代码中抛出了异常，那么这个锁永远都无法释放。



## 未完待续。。。。

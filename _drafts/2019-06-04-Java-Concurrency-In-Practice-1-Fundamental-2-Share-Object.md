---
layout: post
title: Java 并发编程实战-学习日志（一）2：对象的共享
date: 2019-06-04 15:03:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---



<!-- more -->

* 目录
{:toc}
---


# 可见性

可见性，又叫内存可见性（Memory Visibility）。

在单线程下，向某个变量写入值，然后在没有其他写入的情况下，我们总能可以得到相同的值。但是当写和读在不同的线程下进行时，情况却并非如此。

看如下代码：

```java
public class NoVisibility {
    private static boolean ready;
    private static int number;
    private static class ReaderThread extends Thread {
        public void run() {
            while (!ready)
            	Thread.yield();
            System.out.println(number);
        }
    }
    public static void main(String[] args) {
        new ReaderThread().start();
        number = 42;
        ready = true;
    }
}
```

ReaderThread 这个线程可能会一直运行下去，也可能会打印 0 之后退出。这显然不是我们想要的，但产生这个现象的原因是什么呢？这是因为**重排序**。

> 在没有同步的情况下，编译器、处理器以及运行时等都可能对操作的执行顺序进行一些意想不到的调整。在缺乏足够同步的多线程程序中，要想对内存操作的执行顺序进行判断，几乎无法得出正确的结论。



## 1. 失效数据

在没有同步的情况下，线程去读取变量时，可能会得到一个已经失效的值。更糟糕的是，失效值可能不会同时出现：一个线程可能获得某个变量的最新值，而获得另一个变量的失效值。

## 2. 非原子的64位操作

当线程在没有同步的情况下读取变量时，可能会得到一个失效值，但至少这个值是由之前某个线程设置的值，而不是一个随机值。这种安全性保证也被称为**最低安全性**（out-of-thin-air safety）。

最低安全性适用于绝大多数变量，当时存在一个例外：非volatile类型的64位数值变量。Java内存模型要求，变量的读取操作和写入操作都必须是原子操作，但对于非volatile类型的long和double变量，JVM允许将64位的读操作或写操作分解为两个32位的操作。

所以如果对64位的读、写操作在两个线程中，就有可能发生一个写了高32位，一个读了低32位。所以，要用 volatile 关键字来修饰它们，或者加锁。

## 3. 加锁与可见性

内置锁可以用于确保某个线程以一种可预测的方式来查看另一个线程的执行结果。

下图展示了锁是如何保证可见性的：

[![](/images/posts/Visibility_Guarantees_for_Synchronization.png)](/images/posts/Visibility_Guarantees_for_Synchronization.png)

> 加锁的含义不仅仅局限于互斥行为，还包括内存可见性。为了确保所有线程都能看到共享变量的最新值，所有执行读操作或者写操作的线程都必须在同一个锁上同步。

## 4. Volatile变量




## 未完待续。。。

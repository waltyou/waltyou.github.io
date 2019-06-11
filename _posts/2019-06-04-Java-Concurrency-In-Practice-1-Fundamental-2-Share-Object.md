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

Java 提供了一种稍弱的同步机制，即 volatile 变量，用来确保将变量的更新操作通知到其他线程。当把变量声明为volatile类型后，编译器和运行时都会注意到这个变量是**共享**的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile变量不会被缓存在寄存器或者对其他处理器不可见的地方，因此读取volatile类型的变量时总会返回最新写入的值。

从内存可见性的角度看，写入volatile变量相当于退出同步代码块，而读取volatile变量相当于进入同步代码块。

> 仅当 volatile 变量能简化代码的实现时以及对同步策略的验证时，才应该使用它们。

volatile 变量正确的使用方式包括：

- 确保它们自身状态的可见性
- 确保它们所引用对象的状态的可见性
- 标识一些重要的程序生命周期事件的发生

看一个数绵羊睡觉的代码。

```java
volatile boolean asleep；
...
while(!asleep){
	contSomeSheep();
}
```

### 局限性

> 加锁机制既可以保证可见性也可以保证原子性，而volatile变量只能保证可见性。加锁等价于在锁释放的时候，自动将数据进行同步，确保了可见性。

当且仅当满足以下所有条件时，才应该使用 volatile 变量：

- 对变量的写入操作不依赖当前变量的当前值，或者你能确保只有单个线程更新变量的值
- 该变量不会与其他状态变量一起纳入不变性条件中
- 在访问变量时不需要加锁

# 发布与逸出

## 1. 发布

“发布 publish” 一个对象的意思是指，使对象能够在当前作用域之外的代码中使用。

```java
public static Set<Secret> knownSecrets;
public void initialize() {
	knownSecrets = new HashSet<Secret>();
}
```

可以是以下几种情况：

1. 将对象的引用保存到一个公有的静态变量中，以便任何类和线程都能看到该对象；
2. 发布某个对象的时候，在该对象的非私有域中引用的所有对象都会被发布；
3. 发布一个内部的类实例，内部类实例关联一个外部类引用。

## 2. 逸出

“逸出”是指某个不应该发布的对象被公布。

```java
class UnsafeStates {
    private String[] states = new String[] {
    	"AK", "AL" ...
    };
    public String[] getStates() { return states; }
}
```

某个对象逸出后，你必须假设有某个类或线程可能会误用该对象，所以要封装。

不要在构造过程中使this引用逸出。常见错误：在构造函数中启动一个线程。

### 例子

错误使用方式：隐式地使 this 引用逸出

```java
public class ThisEscape {
    public ThisEscape(EventSource source) {
        source.registerListener(
            new EventListener() {
                public void onEvent(Event e) {
                	doSomething(e);
            }
        });
    }
}
```

正确使用方式

```java
public class SafeListener {
    private final EventListener listener;
    
    private SafeListener() {
        listener = new EventListener() {
            public void onEvent(Event e) {
                doSomething(e);
            }
        };
    }
    
    public static SafeListener newInstance(EventSource source) {
        SafeListener safe = new SafeListener();
        source.registerListener(safe.listener);
        return safe;
    }
}
```




## 未完待续。。。

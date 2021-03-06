---
layout: post
title: Java 并发编程实战-学习日志（四）4：Java 内存模型
date: 2020-03-30 18:11:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---


<!-- more -->

---

* 目录
{:toc}
---

## 什么是内存模型及为什么需要它

如果缺少同步，那么将会有许多因素使得线程无法立即甚至永远看到一个线程的操作结果。

- 编译器把变量保存在本地寄存器而不是内存中
- 编译器中生成的指令顺序，可以与源代码中的顺序不同
- 处理器采用乱序或并行的方式来执行指令
- 保存在处理器本地缓存中的值，对于其他处理器是不可见

在单线程中，只要程序的最终结果与在严格串行环境中执行的结果相同，那么上述所有操作都是允许的。

在多线程中，JVM通过同步操作来找出这些协调操作将在何时发生。

JMM规定了JVM必须遵循一组最小保证，这组保证规定了对变量的写入操作在何时将对其他线程可见。



## 1. 平台的内存模型

每个处理器都拥有自己的缓存，并且定期地与主内存进行协调，在不同的处理器架构中提供了不同级别的缓存一致性，即允许不同的处理器在任意时刻从同一个存储位置上看到不同的值。JVM通过在适当的位置上插入**内存栅栏**来屏蔽在JMM与底层平台内存模型之间的差异。**Java程序不需要指定内存栅栏的位置，而只需通过正确地使用同步来找出何时将访问共享状态**。

## 2. 重排序

各种使操作延迟或者看似乱序执行的不同原因，都可以归为重排序，内存级的重排序会使程序的行为变得不可预测。

```java
Thread one = new Thread(new Runnable() {
  public void run() {
    a = 1;
    x = b;
  }
});
```

### 3. Java内存模式简介

Java内存模型是通过各种**操作**来定义的，包括变量的读/写操作，监视器的加锁和释放操作，以及线程的启动和合并操作。

JMM为程序中所有的操作定义了一个**偏序关系**，称为**Happens-Before**，使在正确同步的程序中不存在**数据竞争（缺乏Happens-Before关系，那么JVM可以对它们任意地重排序）**。

- **程序顺序规则**。如果程序中操作A在操作B之前，那么在线程中A操作将在B操作之前执行
- **监视器锁规则。**在监视器锁上的解锁操作必须在同一个监视器锁上的加锁操作之前执行。（显式锁和内置锁在加锁和解锁等操作上有着相同的内存语义） 
- **volatile变量规则。**对volatile变量的写入操作必须在对该变量的读操作之前执行。（原子变量与volatile变量在读操作和写操作上有着相同的语义） 
- **线程启动规则。**在线程上对Thread.start的调用必须在该线程中执行任何操作之前执行
- **线程结束规则。**线程中的任何操作都必须在其他线程检测到该线程已经结束之前执行，或者从Thread.join中成功返回，或者在调用Thread.isAlive时返回false
- **中断规则。**当一个线程在另一个线程上调用interrupt时，必须在被中断线程检测到interrupt调用之前执行（通过抛出InterruptException，或者调用isInterrupted和interrupted）
- **终结器规则。**对象的构造函数必须在启动该对象的终结器之前执行完成
- **传递性。**如果操作A在操作B之前执行，并且操作B在操作C之前执行，那么操作A必须在操作C之前执行。

### 4. 借助同步

”借助（Piggyback）“现有同步机制的可见性属性，对某个未被锁保护的变量的访问操作进行排序（不希望给对象加锁，而又想维护它的顺序）。

Happens-Before排序包括：

- 将一个元素放入一个线程安全容器的操作将在另一个线程从该容器中获得这个元素的操作之前执行
- 在CountDownLatch上的倒数操作将在线程从闭锁上的await方法返回之前执行
- 释放Semaphore许可的操作将在从该Semaphore上获得一个许可之前执行
- Future表示的任务的所有操作将在从Future.get中返回之前执行
- 向Executor提交一个Runnable或Callable的操作将在任务开始执行之前执行
- 一个线程到达CyclicBarrier或Exchange的操作将在其他到达该栅栏或交换点的线程被释放之前执行。如果CyclicBarrier使用一个栅栏操作，那么到达栅栏的操作将在栅栏操作之前执行，而栅栏操作又会在线程从栅栏中释放之前执行

## 发布

造成不正确发布的真正原因："发布一个共享对象"与"另一个线程访问该对象"之间缺少一种Happens-Before的关系。

### 1. 不安全的发布

除了不可变对象以外，使用被另一个线程初始化的对象通常都是不安全的，除非对象的发布操作是在使用该对象的线程开始使用之前执行。

```java
public class UnsafeLazyInitialization {
    private static Object resource;
    public static Object getInstance(){
        if (resource == null){
            resource = new Object(); //不安全的发布
        }
        return resource;
    }
}
```

原因一：线程B看到了线程A发布了一半的对象。

原因二：即使线程A初始化Resource实例之后再将resource设置为指向它，线程B仍可能看到对resource的写入操作将在对Resource各个域的写入操作之前发生。因为线程B看到的线程A中的操作顺序，可能与线程A执行这些操作时的顺序并不相同。

### 2. 安全的发布

例：BlockingQueue的同步机制保证put在take后执行，A线程放入对象能保证B线程取出时是安全的。

借助于类库中现在的**同步容器、使用锁保护共享变量、或都使用共享的volatile类型变量**，都可以保证对该变量的读取和写入是按照happens-before排序的。

happens-before事实上可以比安全发布承诺更强的**可见性与排序性**。

## 3. 安全初始化模式

方式一：加锁保证可见性与排序性，存在性能问题.
```java
public class UnsafeLazyInitialization {
    private static Object resource;
    
    public synchronized static Object getInstance(){
        if (resource == null){
            resource = new Object(); //不安全的发布
        }
        return resource;
    }
}
```
方式二：提前初始化，可能造成浪费资源
```java
public class EagerInitialization {
     private static Object resource = new Object();
     public static Object getInstance(){
         return resource;
     }
 }
```
方式三：延迟初始化，建议

```java
public class ResourceFactory {
    private static class ResourceHolder{
        public static Object resource = new Object();
    }
    public static Object getInstance(){
        return ResourceHolder.resource;
    }
}
```
方式四：双重加锁机制，注意保证volatile类型，否则出现一致性问题

```java
public class DoubleCheckedLocking {
    private static volatile Object resource;
    public static Object getInstance(){
        if (resource == null){
            synchronized (DoubleCheckedLocking.class){
                if (resource == null){
                    resource = new Object(); 
                }
            }
        }
        return resource;
    }
}
```

## 初始化过程中的安全性

- 如果能确保初始化过程的安全性，被正确构造的不可变对象在没有同步的情况下也能安全地在多个线程之间共享
- 如果不能确保初始化的安全性，一些本应为不可变对象的值将会发生改变

初始化安全性只能保证通过final域可达的值从构造过程完成时可见性。对于通过非final域可达的值，或者在构成过程完成后可能改变的值，必须采用同步来确保可见性.



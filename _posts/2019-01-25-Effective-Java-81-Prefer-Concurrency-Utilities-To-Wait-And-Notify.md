---
layout: post
title: 《Effective Java》学习日志（十）81:偏爱使用并发工具来wait和notify
date: 2019-01-25 09:45:04
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

本书的第一版专门用一个项目来正确使用Wait和Notify。它的建议仍然有效，并在本项末尾进行了总结，但这个建议远不如以前那么重要。 这是因为使用wait和notify的原因要少得多。 从Java 5开始，该平台提供了更高级别的并发实用程序，可以执行以前必须在等待和通知时手动编写代码的各种操作。 **鉴于正确使用wait和notify的困难，您应该使用更高级别的并发实用程序。**

java.util.concurrent中的高级实用程序分为三类：Executor Framework，在Item 80中简要介绍了它; 并发集合 concurrent collections; 和同步器 synchronizers。 本项简要介绍了并发集合和同步器。

并发集合是标准集合接口（如List，Queue和Map）的高性能并发实现。 为了提供高并发性，这些实现在内部管理自己的同步（第79项）。 **因此，不可能从并发集合中排除并发活动; 锁定它只会减慢程序**。

因为您不能排除并发集合上的并发活动，所以您也不能以原子方式组合对它们的方法调用。 因此，并发集合接口配备了依赖于状态的修改操作，这些操作将几个原语组合成单个原子操作。 事实证明，这些操作对并发集合非常有用，它们使用默认方法（第21项）添加到Java 8中相应的集合接口中。

例如，Map的putIfAbsent（key，value）方法插入键的映射（如果不存在）并返回与键关联的先前值，如果没有则返回null。 这样可以轻松实现线程安全的规范化映射。 此方法模拟String.intern的行为：

```java
// Concurrent canonicalizing map atop ConcurrentMap - not optimal
private static final ConcurrentMap<String, String> map = new ConcurrentHashMap<>();

public static String intern(String s) {
    String previousValue = map.putIfAbsent(s, s);
    return previousValue == null ? s : previousValue;
}
```

事实上，你可以做得更好。 ConcurrentHashMap针对检索操作进行了优化，例如get。 因此，如果get表明有必要，最初只需调用get并调用putIfAbsent：

```java
// Concurrent canonicalizing map atop ConcurrentMap - faster!
public static String intern(String s) {
    String result = map.get(s);
    if (result == null) {
        result = map.putIfAbsent(s, s);
        if (result == null)
        	result = s;
    } 
    return result;
}
```

除了提供出色的并发性外，ConcurrentHashMap非常快。 在我的机器上，上面的实习方法比String.intern快6倍（但请记住，String.intern必须采用一些策略来防止在长期存在的应用程序中泄漏内存）。 并发集合使同步集合在很大程度上已经过时。 例如，**使用ConcurrentHashMap优先于Collections.synchronizedMap**。 简单地用并发映射替换同步映射可以显着提高并发应用程序的性能。

一些集合接口使用阻塞操作进行扩展，这些操作等待（或阻塞）直到可以成功执行。 例如，BlockingQueue扩展了Queue并添加了几个方法，包括take，它从队列中删除并返回head元素，等待队列为空。 这允许阻塞队列用于工作队列（也称为生产者 - 消费者队列），一个或多个生产者线程将工作项排入其中，并且一个或多个消费者线程从哪个队列变为可用时出列并处理项目。 正如您所期望的那样，大多数ExecutorService实现（包括ThreadPoolExecutor）都使用BlockingQueue（Item 80）。

同步器是使线程能够彼此等待的对象，允许它们协调它们的活动。 最常用的同步器是 CountDownLatch 和Semaphore。 不太常用的是 CyclicBarrier 和 Exchanger。 最强大的同步器是 Phaser。

CountDownLatch 是一次性使用的屏障，允许一个或多个线程等待一个或多个其他线程执行某些操作。 CountDownLatch的唯一构造函数接受一个int，它是在允许所有等待的线程继续之前必须在latch上调用countDown方法的次数。

在这个简单的原语上构建有用的东西是非常容易的。 例如，假设您要构建一个简单的框架来计算操作的并发执行时间。 该框架由一个执行器执行操作的单个方法，一个表示要并发执行的操作数的并发级别以及一个表示该操作的runnable组成。 在计时器线程启动时钟之前，所有工作线程都准备好自己运行操作。 当最后一个工作线程准备好运行该操作时，计时器线程“触发起始枪”，允许工作线程执行操作。 一旦最后一个工作线程完成执行操作，计时器线程就会停止计时。 直接在等待和通知的基础上实现这个逻辑至少可以说是混乱，但在CountDownLatch之上它是令人惊讶的直截了当：

```java
// Simple framework for timing concurrent execution
public static long time(Executor executor, int concurrency,
							Runnable action) throws InterruptedException {
    CountDownLatch ready = new CountDownLatch(concurrency);
    CountDownLatch start = new CountDownLatch(1);
    CountDownLatch done = new CountDownLatch(concurrency);
    
    for (int i = 0; i < concurrency; i++) {
        executor.execute(() -> {
            ready.countDown(); // Tell timer we're ready
            try {
                start.await(); // Wait till peers are ready
                action.run();
            } catch (InterruptedException e) {
            	Thread.currentThread().interrupt();
            } finally {
            	done.countDown(); // Tell timer we're done
            } 
        });
    }
    
    ready.await(); // Wait for all workers to be ready
    long startNanos = System.nanoTime();
    start.countDown(); // And they're off!
    done.await(); // Wait for all workers to finish
    return System.nanoTime() - startNanos;
}
```

请注意，该方法使用三个倒计时锁存器。 第一个就绪，由工作线程用来告诉计时器线程何时准备就绪。 工作线程然后等待第二个锁存器，这是开始。 当最后一个工作线程调用ready.countDown时，计时器线程记录开始时间并调用start.countDown，允许所有工作线程继续。 然后，计时器线程等待第三个锁存器完成，直到最后一个工作线程完成运行并调用done.countDown。 一旦发生这种情况，计时器线程就会唤醒并记录结束时间。

还有一些细节需要注意。传递给time方法的执行程序必须允许创建至少与给定并发级别一样多的线程，否则测试将永远不会完成。这被称为线程饥饿僵局[Goetz06,8.1.1]。如果一个工作线程捕获一个InterruptedException，它会使用成语Thread.currentThread（）。interrupt（）重新断言该中断并从其run方法返回。这允许执行程序在其认为合适时处理中断。请注意，System.nanoTime用于计算活动的时间。**对于间隔计时，请始终使用System.nanoTime而不是System.currentTimeMillis**。 System.nanoTime更准确，更精确，不受系统实时时钟调整的影响。最后，请注意，此示例中的代码不会产生准确的计时，除非操作执行了大量工作，例如一秒钟或更长时间。准确的微基准测试是非常困难的，最好借助于jmh [JMH]等专用框架来完成。

这个项目只涉及使用并发实用程序可以做的事情的表面。 例如，前一个示例中的三个倒计时锁存器可以由单个CyclicBarrier或Phaser实例替换。 结果代码会更简洁，但可能更难理解。

虽然您应该始终优先使用并发实用程序来等待并通知，但您可能必须维护使用wait和notify的旧代码。 wait方法用于使线程等待某些条件。 必须在同步区域内调用它，该区域锁定调用它的对象。 这是使用wait方法的标准习惯用法：

```java
// The standard idiom for using the wait method
synchronized (obj) {
    while (<condition does not hold>)
    	obj.wait(); // (Releases lock, and reacquires on wakeup)
    ... // Perform action appropriate to condition
}
```

**始终使用wait循环模式来调用wait方法; 永远不要在循环之外调用它**。 循环用于测试等待前后的状况。

如果条件已经存在，则在等待之前测试条件并跳过等待以确保活跃。 如果条件已经存在并且在线程等待之前已经调用了notify（或notifyAll）方法，则无法保证线程将从等待中唤醒。

如果条件不成立，等待和等待后再次测试条件是必要的，以确保安全。 如果线程在条件不成立时继续执行操作，则可以销毁由锁保护的不变量。 当条件不成立时，线程可能会唤醒的原因有多种：

- 另一个线程可以获得锁并在线程调用notify和等待线程醒来之间改变了保护状态。
- 当条件不成立时，另一个线程可能会意外或恶意地调用通知。 类通过等待可公开访问的对象来暴露自己这种恶作剧。 在公共可访问对象的同步方法中的任何等待都容易受到此问题的影响。
- 通知线程在唤醒等待线程时可能过于“慷慨”。 例如，即使只有一些等待的线程满足条件，通知线程也可以调用notifyAll。
- 等待线程可以（很少）在没有通知的情况下唤醒。 这被称为虚假唤醒。

相关问题是是否使用notify或notifyAll来唤醒等待线程。 （回想一下，如果存在这样一个线程，notify会唤醒一个等待的线程，并且notifyAll会唤醒所有等待的线程。）有时候你应该总是使用notifyAll。 这是合理的，保守的建议。 它总是会产生正确的结果，因为它可以保证你唤醒需要被唤醒的线程。 您也可以唤醒其他一些线程，但这不会影响程序的正确性。 这些线程将检查它们正在等待的条件，并且发现它为假，将继续等待。

作为优化，如果可能在等待集中的所有线程都在等待相同的条件并且一次只有一个线程可以从条件变为true中受益，则可以选择调用notify而不是notifyAll。

即使满足这些先决条件，也可能有理由使用notifyAll代替通知。 正如将循环中的等待调用置于可公开访问的对象上的意外或恶意通知一样，使用notifyAll代替通知可以防止不相关的线程发生意外或恶意等待。 否则，这样的等待可以“吞下”关键通知，使其预期接收者无限期地等待。

总之，与java.util.concurrent提供的高级语言相比，直接使用wait和notify就像在“并发汇编语言”中编程一样。 **在新代码中很少有（如果有的话）使用wait和notify的理由**。 如果您维护使用wait和notify的代码，请确保它始终使用标准惯用法在while循环内调用wait。 通常应优先使用notifyAll方法进行通知。 如果使用通知，必须非常小心以确保活跃。


---
layout: post
title: 《Effective Java》学习日志（十）84:不要依赖线程调度器
date: 2019-02-12 10:32:00
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

当许多线程可以运行时，线程调度程序会确定哪些线程可以运行以及运行多长时间。 任何合理的操作系统都会尝试公平地做出这个决定，但策略可能会有所不同。 因此，精心编写的程序不应该依赖于策略的细节。 **任何依赖于线程调度程序以获得正确性或性能的程序都可能是不可移植的。**

编写健壮、响应迅速的可移植程序的最佳方法是确保可运行线程的平均数量不会明显大于处理器数量。 这使得线程调度程序几乎没有选择：它只是运行可运行的线程，直到它们不再可运行。 即使在完全不同的线程调度策略下，程序的行为也不会有太大变化。 请注意，可运行线程的数量与线程总数不同，后者可能要高得多。 等待的线程不可运行。

保持可运行线程数量较少的主要技术是让每个线程做一些有用的工作，然后等待更多。 如果线程没有做有用的工作，它们就不应该运行。 就Executor Framework而言（第80项），这意味着适当地调整线程池的大小并保持任务简短但不会太短，或者调度开销会损害性能。

线程不应该忙等待，反复检查等待其状态改变的共享对象。 除了使程序容易受到线程调度程序的变幻莫测之外，忙等待大大增加了处理器的负担，减少了其他人可以完成的有用工作量。 作为不该做的极端例子，请考虑*CountDownLatch*的这种反常重新实现：

```java
// Awful CountDownLatch implementation - busy-waits incessantly!
public class SlowCountDownLatch {
    private int count;
    
    public SlowCountDownLatch(int count) {
        if (count < 0)
        	throw new IllegalArgumentException(count + " < 0");
        this.count = count;
    }
    
    public void await() {
        while (true) {
            synchronized(this) {
                if (count == 0)
                	return;
            }
        }
    }
    
    public synchronized void countDown() {
        if (count != 0)
        	count--;
    }
}
```

在我的机器上，当1000个线程在锁存器上等待时，SlowCountDownLatch比Java的CountDownLatch慢大约十倍。 虽然这个例子看起来有点牵强，但是看到一个或多个线程不必要地运行的系统并不罕见。 性能和可移植性可能会受到影响。

当面对一个因为某些线程没有获得相对于其他线程足够的CPU时间而几乎无法工作的程序时，**要抵制通过调用Thread.yield来“修复”程序的诱惑**。 您可能会成功地使程序在这个方法之后工作，但它不会是可移植的。 在第一个JVM实现上提高性能的相同yield调用，可能会在第二个jvm内变得更糟，在第三个丝毫不起作用。 **Thread.yield没有可测试的语义**。 更好的做法是重构应用程序以减少可并发运行的线程数。

类似警告适用的相关技术是调整线程优先级。 **线程优先级是Java中最不便携的功能之一**。 通过调整一些线程优先级来调整应用程序的响应性并不是不合理的，但它很少是必需的并且它是不可移植的。 尝试通过调整线程优先级来解决严重的活跃度问题是不合理的。 在您找到并解决根本原因之前，问题可能会重新出现。

总之，不要依赖线程调度程序来确定rogram的正确性。 由此产生的程序既不健壮也不便携。 作为推论，不要依赖*Thread.yield*或线程优先级。 这些设施仅仅是调度程序的提示。 可以谨慎地使用线程优先级来提高已经工作的程序的服务质量，但是它们永远不应该用于“修复”几乎不起作用的程序。

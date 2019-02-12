---
layout: post
title: 《Effective Java》学习日志（十）78:同步对共享可变数据的访问
date: 2019-01-15 09:45:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

进入新的一章：并发。

<!-- more -->

------

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

------

## 目录
{:.no_toc}

* 目录
{:toc}

------

**synchronized**关键字确保只有一个线程可以一次执行方法或块。 许多程序员只将同步视为一种互斥方式，以防止一个对象在被另一个线程修改时被一个线程看到处于不一致状态。 在此视图中，对象以一致状态（第17项）创建，并由访问它的方法锁定。 这些方法观察状态并可选地导致状态转换，将对象从一个一致状态转换为另一个状态。 正确使用同步可确保任何方法都不会在不一致的状态下观察对象。

这种观点是正确的，但这只是故事的一半。 如果没有同步，其他线程可能看不到一个线程的更改。 同步不仅阻止线程观察处于不一致状态的对象，而且确保进入同步方法或块的每个线程都能看到由同一个锁保护的所有先前修改的效果。

语言规范保证读取或写入变量是原子的，除非变量的类型为long或double [JLS，17.4,17.7]。 换句话说，读取long或double以外的变量可以保证返回由某个线程存储到该变量中的值，即使多个线程同时修改变量而没有同步也是如此。

您可能听说它说要提高性能，在读取或写入原子数据时应该省去同步。 这个建议是危险的错误。 虽然语言规范保证线程在读取字段时不会看到任意值，但它并不保证一个线程写入的值对另一个线程可见。 **线程之间的可靠通信以及互斥都需要同步**。 这是由于语言规范的一部分称为内存模型，它指定了一个线程所做的更改何时以及如何变得对其他人可见[JLS，17.4; Goetz06,16]。	

即使数据是原子可读和可写的，未能同步访问共享可变数据的后果也可能是可怕的。 考虑从另一个线程停止一个线程的任务。 这些库提供了Thread.stop方法，但是这个方法很久以前就被弃用了，因为它本质上是不安全的 - 它的使用会导致数据损坏。 **不要使用Thread.stop**。 从另一个线程中停止一个线程的推荐方法是让第一个线程轮询一个最初为false的布尔字段，但是第二个线程可以设置为true以指示第一个线程要自行停止。 因为读取和写入布尔字段是原子的，所以一些程序员在访问字段时不需要同步：

```java
// Broken! - How long would you expect this program to run?
public class StopThread {
    private static boolean stopRequested;
    
    public static void main(String[] args)
        	throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested)
                i++;
            });
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}
```

您可能希望此程序运行大约一秒钟，之后主线程将stopRequested设置为true，从而导致后台线程的循环终止。 然而，在我的机器上，程序永远不会终止：后台线程永远循环！

问题是在没有同步的情况下，无法确保后台线程何时（如果有的话）将看到主线程所做的stopRequested值的变化。 在没有同步的情况下，虚拟机将会如此地转换代码：

```java
while (!stopRequested)
	i++;
```

转换为：

```java
if (!stopRequested)
    while (true)
    	i++;
```

这种优化称为 `hoisting` 提升，它正是OpenJDK Server VM所做的。 结果是活跃度失败：程序无法取得进展。 解决此问题的一种方法是同步对stopRequested字段的访问。 正如预期的那样，该程序大约一秒钟终止：

```java
// Properly synchronized cooperative thread termination
public class StopThread {
    private static boolean stopRequested;
    
    private static synchronized void requestStop() {
    	stopRequested = true;
    }
    private static synchronized boolean stopRequested() {
    	return stopRequested;
    }
    public static void main(String[] args)
    		throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested())
                i++;
            });
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        requestStop();
    }
}
```

请注意，write方法（requestStop）和read方法（stop-Requested）都是同步的。 仅同步write方法是不够的！ **除非读取和写入操作同步，否则不保证同步有效**。 偶尔只能同步写入（或读取）的程序似乎可以在某些机器上运行，但在这种情况下，外观是欺骗性的。

即使没有同步，StopThread中的synchronized方法的操作也将是原子的。 换句话说，这些方法的同步仅用于其通信效果，而不是用于互斥。 虽然在循环的每次迭代中同步的成本很小，但是有一个正确的替代方案，其更简洁，其性能可能更好。 如果将stopRequested声明为volatile，则可以省略第二版StopThread中的锁定。 虽然volatile修饰符不执行互斥，但它保证读取该字段的任何线程都将看到最近写入的值：

```java
// Cooperative thread termination with a volatile field
public class StopThread {
    private static volatile boolean stopRequested;
    
    public static void main(String[] args)
    		throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested)
            i++;
            });
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
	}
}
```

使用volatile时必须小心。 考虑以下方法，该方法应该生成序列号：

```java
// Broken - requires synchronization!
private static volatile int nextSerialNumber = 0;
public static int generateSerialNumber() {
    return nextSerialNumber++;
}
```

该方法的目的是保证每次调用都返回一个唯一值（只要不超过2^32次调用）。 方法的状态由单个可原子访问的字段nextSerialNumber组成，该字段的所有可能值都是合法的。 因此，不需要同步来保护其不变量。 但是，如果没有同步，该方法将无法正常工作。

问题是增量运算符（++）不是原子的。 它对nextSerialNumber字段执行两个操作：首先它读取值，然后它写回一个新值，等于旧值加1。 如果第二个线程在线程读取旧值并写回新值之间读取字段，则第二个线程将看到与第一个线程相同的值并返回相同的序列号。 这是安全故障：程序计算错误的结果。

修复generateSerialNumber的一种方法是将synchronized修饰符添加到其声明中。 这确保了多个调用不会交错，并且每次调用该方法都会看到所有先前调用的效果。 完成后，您可以并且应该从nextSerialNumber中删除volatile修饰符。 要防止该方法，请使用long而不是int，或者如果nextSerialNumber即将换行则抛出异常。

更好的是，遵循第59条中的建议并使用类AtomicLong，它是`java.util.concurrent.atomic`的一部分。 该包提供了对单个变量进行无锁，线程安全编程的原语。 虽然volatile只提供同步的通信效果，但这个包也提供了原子性。 这正是我们想要的generateSerialNumber，它可能胜过同步版本：

```java
// Lock-free synchronization with java.util.concurrent.atomic
private static final AtomicLong nextSerialNum = new AtomicLong();

public static long generateSerialNumber() {
	return nextSerialNum.getAndIncrement();
}
```

避免此项中讨论的问题的最佳方法是不共享可变数据。 共享不可变数据（第17项）或根本不共享。 换句话说，**将可变数据限制在单个线程中**。 如果采用此策略，则必须对其进行记录，以便在程序发展时维护策略。 深入了解您正在使用的框架和库也很重要，因为它们可能会引入您不知道的线程。

一个线程可以修改数据对象一段时间然后与其他线程共享它，只同步共享对象引用的行为。 然后，其他线程可以在不进一步同步的情况下读取对象，只要它不再被修改即可。 据说这些物体实际上是不可改变的[Goetz06,3.5.4]。 将这样的对象引用从一个线程转移到其他线程称为安全发布[Goetz06,3.5.3]。 有许多方法可以安全地发布对象引用：您可以将它作为类初始化的一部分存储在静态字段中; 您可以将其存储在易失性字段，最终字段或通过正常锁定访问的字段中; 或者你可以将它放入并发集合中（第81项）。

总之，**当多个线程共享可变数据时，每个读取或写入数据的线程都必须执行同步**。 在没有同步的情况下，无法保证一个线程的更改对另一个线程可见。 未能同步共享可变数据的处罚是活跃性和安全性故障。 这些失败是最难调试的。 它们可以是间歇性的和时间相关的，并且程序行为可以从一个VM到另一个VM发生根本变化。 如果您只需要线程间通信，而不是互斥，则volatile修饰符是可接受的同步形式，但正确使用可能很棘手。

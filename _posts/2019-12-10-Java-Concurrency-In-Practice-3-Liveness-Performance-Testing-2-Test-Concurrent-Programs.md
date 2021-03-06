---
layout: post
title: Java 并发编程实战-学习日志（三）3：并发程序的测试
date: 2019-12-10 18:01:04
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

## 前言

在编写并发程序时，可以采用与写串行程序时相同的设计原则与设计模式。二者的差异在于，并发程序存在一定程度的不确定性。

并发测试大致可以分为两类：

- 安全性测试
- 活跃性测试
  - 进展测试
  - 无进展测试

与活跃性测试相关的是**性能测试**。性能可以通过多个方面来衡量，包括：

- 吞吐量
- 响应性
- 可伸缩性



## 正确性测试

在为某个并发类设计单元测试时，首先需要执行与测试串行类时相同的分析——找出需要检查的不变性条件和后验条件.

### 1. 基本的单元测试

这部分的测试代码与并发性没多大关系，用JUnit直接写测试代码就可以了。

### 2. 对阻塞操作的测试

如果在某个测试用例创建的辅助线程中发现了一个错误，那么框架通常无法得知与这个线程相关的是哪一个测试，所以需要通过一些工作将成功或失败信息传递回主线程，从而才能将相应的信息报告出来。

### 3. 安全性测试

编写的测试程序本身就是一个并发程序，测试在数据竞争条件下是否会发生错误。这就需要一个并发的测试程序。或许比编写本身要测试的类更加困难。

通过一个对顺序敏感的校验和计算函数来计算 所有入列的元素以及出列元素的检验和，并进行比较。如果二者相等，那么测试就是成功的。如果只有一个生产者一个消费者，那么 这种方法能发挥最大的作用。因为它不仅能测试出是否取出了正确的元素，而且还能测试出元素被取出的顺序是否正确。

如果要将这种方法扩展到多生产者多消费者的情况时，就需要一个对元素入列和出列顺序不敏感的校验和函数。从而在测试程序运行完以后，可以将多个检验和以不同的顺序组合起来，如果不是这样，多个线程就需要访问 同一个共享的检验和变量 ，因此就需要同步，这将成为一个并发的瓶颈。

要确保测试程序能够正确地测试所有要点，就一定不能让编译器可以预先猜测到检验 和的值。那么会对许多 其他 的测试造成影响。由于大多数随机类生成器都是线程安全的。并且会带来额外的同步开销。所以还不如用一个简单的伪随机函数 。

这种测试应该放在多处理器的系统上运行。要最大程序地检测出一些对执行时序敏感的数据竞争，那么测试中的线程数量应该多于CPU数量，这样在任意时刻都会有一些线程在运行，而另一些被交换出去，从而可以检查线程是交替行为的可预测性。

### 4. 资源管理的测试

测试的另一个方面就是要判断类中是否没有做它不应该做的事情，例如资源泄漏。对于任何持有或管理其他对象的对象，都应该在不需要这些对象时销毁对它们的引用。这种存储资源泄漏不仅会妨碍垃圾回收器回收内存（或者是线程、文件句柄、套接字、数据库连接或其他有限资源），而且还会导致资源耗尽以及应用程序失败。所以要限制缓存的大小。

通过一些测量应用程序中内存使用情况的堆检查工具，可以很容易地测试出对内存的不合理占用，许多商业和开源的堆分析工具中都支持这种功能。

### 5. 使用回调

在构造测试案例时，对客户提供的代码进行回调是非常有帮助的。回调函数的执行通常是在对象生命周期的一些已知位置上，并且在这些位置上非常适合判断不变性条件是否被破坏。例如，在ThreadPoolExecutor中将调用任务的Runnable和ThreadFactory。

在测试线程池时，需要测试执行策略的多个方面：在需要更多线程时创建新线程，在不需要时不创建，以及当需要回收空闲线程时执行回收操作等。要构造一个全面的测试方案是很困难的，但其中许多方面的测试都可以单独进行。

通过使用自定义的线程工厂，可以对线程的创建过程进行控制。

### 6. 产生更多的交替操作

由于并发代码中发生的错误一般都是低概率事件，所以在测试并发错误时需要反复地执行许多次，但有些方法可以提高发现这些错误的概率 ，在前面提到过，在多处理器系统上，如果 处理器的数量少于活动线程的数量，那么 与单处理器的系统 或者 包含多个处理器的系统相比，将能产生更多的交替行为。同样，如果在不同的处理器数量、操作系统以及处理器架构的系统上进行测试，就可以发现那些在特定运行环境中才会出现的问题。

有一种有用的方法能提高交替操作的数量。以便能更有效的搜索程序的状态空间：在访问共享状态的操作中，使用**Thred.yield**将产生更多的上下文切换。当代码在访问状态的时候没有使用足够的同步，将存在一些对执行时序敏感的错误，通过在某个操作的执行过程 中调用yield方法，可以将这些错误暴露出来。这种方法需要在测试中添加一些调用并且在正式产品中删除这些调用。

```java
public synchronized void tranferCredits(Account from,Account to,int amount) {  
    from.setBalance(from.getBalance()-amount);  
    if (random.nextInt(1000)>THRESHOLD) {  
        Thread.yield();  
    }  
    to.setBalance(to.getBalance()+amount);  
}
```



## 性能测试

性能测试要符合当前程序的应用场景，理想情况下应该反映出被测试对象的在应用程序中的实际用法。

性能测试将衡量典型测试用例中的端到端功能。通常，要获得一组合理的使用场景并不容易，理想情况下，在测试中应该反映出被测试对象在应用程序中的实际用法。

第二个目标就是根据经验值来调整各种不同的限值，例如线程数量，缓存容量等等，这些限值都依赖于平台特性（例如，处理器的类型、处理器的步进级别、CPU的数量或内存大小等），因此需要动态地进行配置，我们通常要合理的选择这些值，使程序能够在更多的系统上良好的运行。

### 1. 在PutTakeTest中增加计时功能

之前对PutTakeTest的主要扩展就是测试运行一次所需要的时间。现在我们不测量单个操作的时间，而是实现一种更精确的测量方式：记录整个运行过程的时间，然后除以总操作的数量，从而得到每次操作的运行时间。之前使用了CyclicBarrier来启动和结束工作者线程，因此可以对其进行扩展：使用一个栅栏动作来测量启动和结束时间。

### 2. 多种算法的比较

测试结果表明，LinkedBlockgingQueue的可伸缩性要高于ArrayBlockingQueue，初看起来这个结果似乎有些奇怪，链表队列在每次插入元素时，都必须分配一个链表节点对象，这似乎比基于数组的队列相比，链表队列的put和take等方法支持并发性更高的访问，因为一些优化后的链接队列算法能将队列头节点的更新操作与尾节点的更新操作分享开来。因此如果算法能通过多执行一些内存分配操作来降低竞争程度，那么这种算法通常具有更高的可伸缩性。

### 3. 响应性衡量

在前面讨论过，如果缓存过小，那么将导致非常多的上下文切换次数，这即使在非公平模式中也会导致很低的吞吐量，因此在几乎每个操作中都会执行上下文切换。

如果线程由于密集的同步需求而被持续地阻塞，否则非公平的信号量通常能实现更好的吞吐量，而公平的信号量则实现更低的变动性。因为这些结果之间的差异非常大，所以Semaphore要求客户选择针对哪一个特性进行优化。

---

## 避免性能测试的陷阱

你必须提防多种编码陷阱，它们会使性能测试变得毫无意义。

### 1. 垃圾回收

有两种策略可以防止垃圾回收操作对测试结果产生偏差。第一种策略是，确保垃圾回收操作在测试运行的整个期间都不会执行（可以在调用JVM时指定-verbose：gc来判断是否执行了垃圾回收操作）。第二种策略是，确保垃圾回收操作在测试期间执行多次，这样测试程序就能充分反映出运行期间的内存分配与垃圾回收等开销。通常第二策略更好，它要求更长的测试时间，并且更有可能反映实际环境下的性能。

### 2. 动态编译

有一种方式可以防止动态编译对测试结果产生偏差，就是使程序运行足够长的时间（至少数分钟），这样编译过程以及解释执行都只是总运行时间的很小一部分。另一种方法是使代码预先运行一段时间并且不测试这段时间内的代码性能，这样在开始计时前代码就已经被完全编译了。在HotSpot中，如果在运行程序时使用命令命令行选项-xx:+PrintCompilation,那么当动态编译运行时将输出一条信息，你可以通过这条消息来验证动态编译是在测试运行前，而不是在运行过程中。

通过在同一个JVM中将相同的测试运行多次，可以验证测试方法的有效性。第一组结果应该作为“预先执行”的结果而丢弃，如果在剩下的结果中任然存在不一致的地方，那么就需要进一步对测试进行分析，从而找出结果不可重复的原因。

JVM会使用不同的后台线程来执行辅助任务。当在单次运行中测试多个不相关的计算密集性操纵时，一种好的做法是在不同操作的测试之间插入显式的暂停，从而使JVM能够与后台任务保持步调一致，同时将被测试任务的干扰降至最低。（然而，当测量多个相关操作时，例如将相同测试运行多次，如果按照这种方式来排除JVM后台任务，那么可能会得出不真实的结果）。


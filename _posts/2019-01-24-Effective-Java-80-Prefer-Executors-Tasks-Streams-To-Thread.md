---
layout: post
title: 《Effective Java》学习日志（九）80:比起Thread，偏爱executor、tasks、Streams
date: 2019-01-24 15:45:04
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

本书的第一版包含一个简单工作队列的代码[Bloch01，第49项]。 此类允许客户端将后台线程的异步处理工作排入队列。 当不再需要工作队列时，客户端可以调用一个方法，要求后台线程在完成队列中已有的任何工作后正常终止自身。 实现只不过是一个玩具，但即便如此，它还需要一整页精细，细致的代码，如果你没有恰到好处的话，这种代码很容易出现安全和活动失败。 幸运的是，没有理由再编写这种代码了。

到本书第二版出版时，java.util.concurrent已添加到Java中。 该软件包包含一个Executor Framework，它是一个灵活的基于接口的任务执行工具。 创建一个比本书第一版更好的工作队列只需要一行代码：

```java
ExecutorService exec = Executors.newSingleThreadExecutor();
```

以下是如何提交runnable执行：

```java
exec.execute(runnable);
```

这里是如何告诉执行程序优雅地终止（如果你没有这样做，很可能你的VM不会退出）：

```java
exec.shutdown();
```

您可以使用执行程序服务执行更多操作。 例如，您可以等待特定任务完成（使用get方法，如第79页的第79项所示），您可以等待任何或所有任务集合完成（使用invokeAny或invokeAll方法） ，您可以等待执行程序服务终止（使用awaitTermination方法），您可以在完成任务时逐个检索任务结果（使用ExecutorCompletionService），您可以安排任务在特定时间运行或定期运行 （使用ScheduledThreadPoolExecutor），依此类推。

如果您希望多个线程处理来自队列的请求，只需调用另一个静态工厂，该工厂创建一种称为线程池的不同类型的执行器服务。 您可以创建具有固定或可变数量线程的线程池。 java.util.concurrent.Executors类包含静态工厂，它们提供了您需要的大多数执行程序。 但是，如果您想要一些与众不同的东西，可以直接使用ThreadPoolExecutor类。 此类允许您配置线程池操作的几乎每个方面。

为特定应用程序选择执行程序服务可能很棘手。 对于小程序或负载较轻的服务器，Executors.newCachedThreadPool通常是一个不错的选择，因为它不需要配置，通常“做正确的事情。”但是对于负载很重的生产服务器来说，缓存的线程池不是一个好的选择！ 在缓存的线程池中，提交的任务不会排队，而是立即传递给线程执行。 如果没有可用的线程，则创建一个新线程。 如果服务器负载过重以至于所有CPU都被充分利用并且更多任务到达，则会创建更多线程，这只会使事情变得更糟。 因此，在负载很重的生产服务器中，最好使用Executors.newFixedThreadPool，它为您提供具有固定线程数的池，或直接使用ThreadPoolExecutor类，以实现最大程度的控制。

您不仅应该避免编写自己的工作队列，而且通常应该避免直接使用线程。 当您直接使用线程时，线程既可以作为工作单元，也可以作为执行它的机制。 在执行程序框架中，工作单元和执行机制是分开的。 关键的抽象是工作单元，这是任务。 有两种任务：Runnable及其近亲，Callable（类似于Runnable，除了它返回一个值并且可以抛出任意异常）。 执行任务的一般机制是执行程序服务。 如果您考虑任务并让执行程序服务为您执行它们，您可以灵活地选择适当的执行策略以满足您的需求，并在需求发生变化时更改策略。 本质上，Executor Framework执行Collections Framework为聚合所做的工作。

在Java 7中，Executor Framework被扩展为支持fork-join任务，这些任务由称为fork-join池的特殊执行器服务运行。 由ForkJoinTask实例表示的fork-join任务可以拆分为较小的子任务，而包含ForkJoinPool的线程不仅处理这些任务，而且还“彼此”窃取“任务”以确保所有线程都保持忙碌，从而导致更高的任务 CPU利用率，更高的吞吐量和更低的延迟。 编写和调优fork-join任务很棘手。 并行流（第48项）写在fork连接池的顶部，允许您轻松地利用它们的性能优势，假设它们适合于手头的任务。

对Executor Framework的完整处理超出了本书的范围，但感兴趣的读者可以参考Java Concurrency in Practice。


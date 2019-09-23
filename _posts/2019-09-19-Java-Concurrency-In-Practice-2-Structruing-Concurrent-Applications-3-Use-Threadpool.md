---
layout: post
title: Java 并发编程实战-学习日志（二）3：线程池的使用
date: 2019-09-19 18:45:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

这一部分将介绍对线程池进行配置与调优的一些高级选项，并分析在使用任务执行框架时需要注意的各种危险， 以及一些使用 Executor的高级示例。

<!-- more -->

---

* 目录
{:toc}
---

# 在任务与执行策略之间的隐性耦合

Executor框架可以将任务的提交与任务的执行策略解耦开来。但就像许多对复杂过程的解耦操作那样，这种论断多少有些言过其实了。 虽然Executor框架为制定和修改执行策略都提供了相当大的灵活性， 但**并非所有的任务都能适用所有的执行策略**。 有些类型的任 务需要明确地指定执行策略， 包括：

1. **依赖性任务**。 大多数行为正确的任务都是独立的： 它们不依赖于其他任务的执行时序、 执行结果或其他效果。 当在线程池中执行独立的任务时， 可以随意地改变线程池的大小和配置，这些修改只会对执行性能产生影响。 然而，如果提交给线程池的任务需要依赖其他的任务， 那么就隐含地给执行策略带来了约束， 此时必须小心地维持次些执行策略以避免产生活跃性问题。

2. **使用线程封闭机制的任务**。 与线程池相比， 单线程的Executor能够对并发性做出更强的承诺。 它们能确保任务不会并发地执行， 使你能够放宽代码对线程安全的要求。 对象可以封闭在 任务线程中， 使得在该线程中执行的任务在访问该对象时不需要同步， 即使这些资源不是线程 安全的也没有问题。 这种情形将在任务与执行策略之间形成隐式的耦合——任务要求其执行所在的Executor是单线程的。如果将Executor从单线程环境改为线程池环境， 那么将会失去线程安全性。

3. **对响应时间敏感的任务**。GUI应用程序对于响应时间是敏感的：如果用户在点击按钮后需要很长延迟才能得到可见的反馈， 那么他们会感到不满。如果将一个运行时间较长的任务提交到单线程的Executor中， 或者将多个运行时间较长的任务提交到一个只包含少量线程的线程池 中， 那么将降低由该Executor管理的服务的响应性。

4. **使用ThreadLocal的任务**。ThreadLocal使每个线程都可以拥有某个变量的一个私有“版本“。然而，只要条件允许，Executor可以自由地重用这些线程。在标准的Executor实现中，当执行需求较低时将回收空闲线程，而当需求增加时将添加新的线程，并且如果从任务中抛出了一个未检查异常，那么将用一个新的工作者线程来替代抛出异常的线程。只有当线程本地值 的生命周期受限于任务的生命周期时，在线程池的线程中使用ThreadLocal才有意义，而在线 程池的线程中不应该使用 ThreadLocal在任务之间传递值。

只有当任务都是同类型的并且相互独立时，线程池的性能才能达到最佳。如果将运行时间较长的与运行时间较短的任务混合在一起，那么除非线程池很大，否则将可能造成 “拥塞 ”。如果提交的任务依赖于其他任务，那么除非线程池无限大，否则将可能造成死锁。幸运的是， 在基于网络的典型股务器应用程序中一一网页服务器、邮件服务器以及文件服务器等，它们的 请求通常都是同类型的并且相互独立的。

> 在一些任务中，需要拥有或排除某种特定的执行策略。如果某些任务依赖于其他的任务，那么会要求线程池足够大，从而确保它们依赖任务不会被放入等待队列中或被拒绝，而采用线程封闭机制的任务需要串行执行。通过将这些需求写入文档，将来的代码维护人员就不会由于使用了某种不合适的执行策略而破坏安全性或活跃性。

## 1. 线程饥饿死锁

在线程池中，如果任务依赖于其他任务，那么可能产生死锁。在单线程的Executor中，如 果一个任务将另一个任务提交到同一个Executor,并且等待这个被提交任务的结果，那么通常会引发死锁。第二个任务停留在工作队列中，并等待第一个任务完成，而第一个任务又无法完 成，因为它在等待第二个任务的完成。在更大的线程池中, 只要线程池中的任务需要无限期地等待一些必须由池中其他任务才能提供的资源或条件，例如某个任务等待另一个任务的返回值或执行结果，那么除非线程池足够大，否则将发生线程饥饿死锁。 

以下是线程饥饿死锁的示例。

```java
public class ThreadDeadlock {
    ExecutorService exec = Executors.newSingleThreadExecutor();

    public class LoadFileTask implements Callable<String> {
        private final String fileName;

        public LoadFileTask(String fileName) {
            this.fileName = fileName;
        }

        public String call() throws Exception {
            // Here's where we would actually read the file
            return "";
        }
    }

    public class RenderPageTask implements Callable<String> {
        public String call() throws Exception {
            Future<String> header, footer;
            header = exec.submit(new LoadFileTask("header.html"));
            footer = exec.submit(new LoadFileTask("footer.html"));
            String page = renderBody();
            // Will deadlock -- task waiting for result of subtask
            return header.get() + page + footer.get();
        }

        private String renderBody() {
            // Here's where we would actually render the page
            return "";
        }
    }
}
```

> 每当提交了一个有依赖性 Executor 任务时，要清楚地知道可能会出现线程“饥饿”死锁，因此需要在代码或配置Executor的配置文件中记录线程池的大小限制或配置限制。

除了在线程池大小上的显式限制外， 还可能由于其他资源上的约束而存在一些隐式限制。如果应用程序使用一个包含10个连接的JDBC连接池， 并且每个任务需要一个数据库连接， 那么线程池就好像只有10个线程， 因为当超过10个任务时， 新的任务需要等待其他任务释放连接。







## 未完待续。。。
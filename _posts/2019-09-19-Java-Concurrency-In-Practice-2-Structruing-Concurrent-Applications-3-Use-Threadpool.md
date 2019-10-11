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

## 2. 运行时间较长的任务

如果任务阻塞的时间过长， 那么即使不出现死锁， 线程池的响应性也会变得糟糕。执行时间较长的任务不仅会造成线程池堵塞， 甚至还会增加执行时间较短任务的服务时间。如果线程池中线程的数量远小于在稳定状态下执行时间较长任务的数量， 那么到最后可能所有的线程都会运行这些执行时间较长的任务， 从而影响整体的响应性。

有一项技术可以缓解执行时间较长任务造成的影响， 即限定任务等待资源的时间， 而不要无限制地等待。在平台类库的大多数可阻塞方法中， 都同时定义了限时版本和无限时版本， 例如 Thread.join、BlockingQueue.put、CountDownLatch.await 以及 Selector.select 等。如果等待超时，那么可以把任务标识为失败，然后中止任务或者将任务重新放回队列以便随后执行。这样， 无论任务的最终结果是否成功， 这种办法都能确保任务总能继续执行下去， 并将线程释放 出来以执行一些能更快完成的任务。如果在线程池中总是充满了被阻塞的任务， 那么也可能表明线程池的规模过小。



---

# 设置线程池的大小

线程池的理想大小取决于被提交任务的类型以及所部署系统的特性。在代码中通常不会固定线程池的大小， 而应该通过某种配置机制来提供，或者根据Runtime.availableProcessors来动态计算。

幸运的是， 要设置线程池的大小也并不困难， 只需要避免 “过大” 和 “过小” 这两种极端情况。如果线程池过大，那么大量的线程将在相对很少的CPU和内存资源上发生竞争， 这不仅会导致更高的内存使用量， 而且还可能耗尽资源。如果线程池过小， 那么将导致许多空闲的处理器无法执行工作，从而降低吞吐率。

要想正确地设置线程池的大小，必须分析计算环境、资源预算和任务的特性。在部署的系统中有多少个CPU? 多大的内存？任务是计算密集型、I/0密集型还是二者皆可？它们是否需要像JDBC连接这样的稀缺资源？如果需要执行不同类别的任务，井且它们之间的行为相差很大，那么应该考虑使用多个线程池，从而使每个线程池可以根据各自的工作负载来调整。

对于计算密集型的任务，在拥有N个处理器的系统上，当线程池的大小为N+1时，通常能实现最优的利用率。（即使当计算密集型的线程偶尔由于页缺失故障或者其他原因而暂停时，这个 “额外 ” 的线程也能确保CPU的时钟周期不会被浪费。）要正确地设置线程 池的大小，你必须估算出任务的等待时间与计算时间的比值。这种估算不需要很精确，并且可 以通过一些分析或监控工具来获得。你还可以通过另一种方法来调节线程池的大小：在某个基准负载下，分别设置不同大小的线程池来运行应用程序，并观察CPU利用率的水平。

[![](/images/posts/SetThreadPoolSize.webp)](/images/posts/SetThreadPoolSize.webp)



---

# 配置 ThreadPoolExecutor

ThreadPoolExecutor为一些Executor提供了基本的实现，这些Executor是由 Executors 中 的newCachedThreadPool、newFixedThreadPool和newScheduledThreadExecutor等工厂方法返回的。

## 1. 线程的创建与销毁

线程池的基本大小(Core Pool Size)、最大大小(Maximum Pool Size)以及存活时间等因素共同负责线程的创建与销毁。 基本大小也就是线程池的目标大小， 即在没有任务执行时线程池的大小， 并且只有在工作队列满了的情况下才会创建超出这个数字的线程。 线程池的最大大小表示可同时活动的线程数最的上限。 如果某个线程的空闲时间超过了存活时间， 那么将被标记为可回收的， 并且当线程池的当前大小超过了基本大小时， 这个线程将被终止。

通过调节线程池的基本大小和存活时间， 可以帮助线程池回收空闲线程占有的资源，从而使得这些资源可以用于执行其他工作。（显然， 这是种折衷： 回收空闲线程会产生额外的延迟， 因为当需求增加时， 必须创建新的线程来满足需求。）

`newFixedThreadPool` 工厂方法将线程池的基本大小和最大大小设置为参数中指定的值， 而且创建的线程池不会超时。newCachedThreadPool工厂方法将线程池的最大大小设置为Integer. MAX_VALUE, 而将基本大小设置为零， 并将超时设置为1分钟， 这种方法创建出来的线 程池可以被无限扩展， 并且当需求降低时会自动收缩。其他形式的线程池可以通过显式的 ThreadPoolExecutor构造函数来构造。



## 2. 管理队列任务

在有限的线程池中会限制可并发执行的任务数量。（单线程的Executor是一种值得注意的特例：它们能确保不会有任务并发执行， 因为它们通过线程封闭来实现线程安全性。）

如果无限制地创建线程， 那么将导致不稳定性， 并通过采用固定大小的线程池（而不是每收到一个请求就创建一个新线程 ）来解决这个问题。然而，这个方案并不完整。在高负载情况下， 应用程序仍可能耗尽资源， 只是出现问题的概率较小。 如果新请求的到达速率超过了线程池的处理速率， 那么新到来的请求将累积起来。在线程池中，这些请求会在一个由 Executor管理的 Runnable 队列中等待，而不会像线程那样去竞争 CPU资源。 通过 一个 Runnable 和一个链表节点来表现一个等待中的任务， 当然比使用线程来表示的开销低很多， 但如果客户提交给服务器请求的速率超过了服务器的处理速率，那么仍可能会耗尽资源。

即使请求的平均到达速率很稳定， 也仍然会出现请求突增的情况。 尽管队列有助于缓解任 务的突增问题，但如果任务持续高速地到来，那么最终还是会抑制请求的到达率以避免耗尽内存。甚至在耗尽内存之前，响应性能也将随着任务队列的增长而变得越来越糟。

ThreadPoolExecutor 允许提供一个 BlockingQueue 来保存等待执行的任务。 基本的任务排队方法有 3 种： 无界队列、有界队列和同步移交 (Synchronous Handoff)。

### 无界队列

newFixedThreadPool 和 newSingleThreadExecutor在默认情况 下将使用一个无界的 LinkedBlockingQueue。如果所有工作者线程都处于忙碌状态， 那么任务将在队列中等候。如果任务持续快速地到达， 并且超过了线程池处理它们的速度， 那么队列将无限制地增加。

### 有界队列

 一种更稳妥的资源管理策略是使用有界队列，例如 ArrayBlockingQueue、有界的LinkedBlockingQueue、 PriorityBlockingQueue。有界队列有助于避免资源耗尽的情况发生， 但它又带来了新的问题： 当队列填满后，新的任务该怎么办？在使用有界的工作队列时，队列的大小与线程池的大小必须一起调节。如果线程池较小而队列较大，那么有助于减少内存使用量，降低 CPU 的使用率，同时还可以减少上下文切换， 但付出的代价是可能会限制吞吐量。

### 同步移交

对于非常大的或者无界的线程池，可以通过使用 SynchronousQueue 来避免任务排队， 以及直接将任务从生产者移交给工作者线程。 SynchronousQueue 不是一个真正的队列， 而是一 种在线程之间进行移交的机制。 要将一个元素 放入 SynchronousQueue 中， 必须有另一个线程正在等待接受这个元素。如果没有线程正在等待， 并且线程池的当前大小小于最大值， 那么ThreadPoolExecutor 将创建一个新的线程， 否则根据饱和策略， 这个任务将被拒绝。 使用直接移交将更高效， 因为任务会直接移交给执行它的线程， 而不是被首先放在队列中，然后由工作者线程从队列中提取该任务。 只有当线程池是无界的或者可以拒绝任务时， SynchronousQueue才有实际价值。在 newCachedThreadPool 工厂方法中就使用了 SynchronousQueue。

当使用像 LinkedBlockingQueue 或 ArrayBlockingQueue 这样的 FIFO(先进先出）队列时， 任务的执行顺序与它们的到达顺序相同。如果想进一步控制任务执行顺序， 还可以使用PriorityBlockingQueue, 这个队列将根据优先级来安排任务。任务的优先级是通过自然顺序或Comparator (如果任务实现了Comparable)来定义的。

只有当任务相互独立时， 为线程池或工作队列设置界限才是合理的。如果任务之间存在依赖性， 那么有界的线程池或队列就可能导致线程 ” 饥饿” 死锁问题。此时应该使用无界的线程 池， 例如newCachedThreadPool.

## 3. 饱和策略

当有界队列被填满后， 饱和策略开始发挥作用。ThreadPoolExecutor 的饱和策略可以通过调用setRejectedExecutionHandler 来修改。（如果某个任务被提交到一个巳被关闭的Executor时，也会用到饱和策略。） JDK提供了几种不同的RejectedExecutionHandler实现，每种实现都包含有不固的饱和策略： AbortPolicy、CallerRunsPolicy、DiscardPolicy和DiscardOldestPolicy。

"中止(Abort)"策略是默认的饱和策略，该策略将抛出未检查的RejectedExecution -Exception。调用者可以捕获这个异常，然后根据需求编写自己的处理代码。

当新提交的任务法保存到队列中等待执行时，“抛弃(Discard)"策略会悄悄抛弃该任务。 

“抛弃最旧的（ Discard-Oldest)"策略则会抛弃下一个将被执行的任务， 然后尝试重新提交新的任务。（如果工作队列是一个优先队列， 那么 “抛弃最旧的” 策略将导致抛弃优先级最高的任务， 因此最好不要将 “抛弃最旧的＂ 饱和策略和优先级队列放在一起使用。）

 “调用者运行(Caller-Runs)"策略实现了一种调节机制， 该策略既不会抛弃任务， 也不会抛出异常， 而是将某些任务回退到调用者， 从而降低新任务的流量。 它不会在线程池的某个线程中执行新提交的任务， 而是在一个调用了execute的线程中执行该任务。 我们可以将WebServer示例修改为使用有界队列和 “ 调用者运行” 饱和策略， 当线程池中的所有线程都被占用， 并且工作队列被填满后， 下一个任务会在调用execute 时在主线程中执行门 由于执行任 务需要一定的时间， 因此主线程至少在一段时间内不能提交任何任务， 从而使得工作者线程有时间来处理完正在执行的任务。在这期间， 主线程不会调用accept, 因此到达的请求将被保存 在TCP层的队列中而不是在应用程序的队列中。 如果持续过载， 那么TCP层将最终发现它的 请求队列被填满， 因此同样会开始抛弃请求。 当服务器过载时， 这种过载情况会逐渐向外蔓延开来－从线程池到工作队列到应用程序再到TCP层， 最终达到客户端， 导致服务器在高负载下实现一种平缓的性能降低。

当创建 Executor 时， 可以选择饱和策略或者对执行策略进行修改。 

当工作队列被填满后， 没有预定义的饱和策略来阻塞 execute 。然而， 通过使用 Semaphore（信号量）来限制任务的到达率，就可以实现这个功能。

```java
public class BoundedExecutor {
    private final Executor exec;
    private final Semaphore semaphore;

    public BoundedExecutor(Executor exec, int bound) {
        this.exec = exec;
        this.semaphore = new Semaphore(bound);
    }

    public void submitTask(final Runnable command)
            throws InterruptedException {
        semaphore.acquire();
        try {
            exec.execute(new Runnable() {
                public void run() {
                    try {
                        command.run();
                    } finally {
                        semaphore.release();
                    }
                }
            });
        } catch (RejectedExecutionException e) {
            semaphore.release();
        }
    }
} 
```








## 未完待续。。。
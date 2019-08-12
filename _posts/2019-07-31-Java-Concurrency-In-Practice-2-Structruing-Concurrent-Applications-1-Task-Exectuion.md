---
layout: post
title: Java 并发编程实战-学习日志（二）1：任务执行
date: 2019-07-31 20:13:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

大多数并发应用程序都是围绕“任务执行”来构造的：任务通常是一些抽象且离散的工作单元。

<!-- more -->

---

* 目录
{:toc}
---

# 在线程中执行任务

当围绕任务执行来设计应用程序时，第一步就是找出清晰的任务边界。

如果任务是相互独立的，那么有助于实现并发。

## 1. 串行地执行任务

在应用程序中，可以通过多种策略来调度任务，其中最简单的就是在单个线程中串行地执行各项任务。

```java
public class SingleThreadWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            Socket connection = socket.accept();
            handleRequest(connection);
        }
    }

    private static void handleRequest(Socket connection) {
        // request-handling logic here
    }
}
```

简单明了，但却性能低下，因为它每次只能处理一个请求。通常在处理过程中，I/O 操作是比较耗时的，这时候 CPU 其实处于空闲状态，很浪费资源。


## 2. 显示地为任务创建线程

```java
public class ThreadPerTaskWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            final Socket connection = socket.accept();
            Runnable task = new Runnable() {
                public void run() {
                    handleRequest(connection);
                }
            };
            new Thread(task).start();
        }
    }

    private static void handleRequest(Socket connection) {
        // request-handling logic here
    }
}
```

代码如上，但是千万不要这么做，因为可能会创建过多线程，使得程序崩溃。

## 3. 无限制创建线程的不足

1. 线程生命周期的开销非常高
2. 资源消耗
3. 稳定性: 在可创建线程的数量上存在一个限制，这个限制值随着平台的不同而不同并受到多个因素制约，包括JVM启动参数，Thread构造函数中请求的栈大小，以及底层操作系统对线程的限制等。

---

# Executor框架

java.util.concurrent提供了一种灵活的线程池作为Executor框架的一部分。

```java
public interface Executor{
	void execute(Runnable command)
}
```

Executor基于生产者-消费者模式，提交任务的操作相当于生产者，执行任务的线程则相当于消费者。

## 1. 基于Executor的web服务器

如下所示，定义了一个固定长度的线程池。

```java
public class TaskExecutionWebServer {
    private static final int NTHREADS = 100;
    private static final Executor exec
            = Executors.newFixedThreadPool(NTHREADS);

    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            final Socket connection = socket.accept();
            Runnable task = new Runnable() {
                public void run() {
                    handleRequest(connection);
                }
            };
            exec.execute(task);
        }
    }

    private static void handleRequest(Socket connection) {
        // request-handling logic here
    }
}
```



## 2. 执行策略

将任务提交和执行解耦开来，可以为某任务指定和修改执行策略。

在执行策略中包括了“what 、where、When、How” 等方面，包括： 

1. 在什么（what）线程中执行任务？ 
2. 任务按照什么（what）顺序执行（FIFO、FIFO、优先级）？ 
3. 有多少个（how many）任务能并发执行？ 
4. 在队列中有多少个（how many）任务在等待执行？ 
5. 如果系统由于过载而需要绝句一个任务，那么应该选择（which）哪个任务？另外，如何通（how）知应用程序有任务被拒绝？ 
6. 在执行一个任务之前或之后，应该进行哪些（what）动作？



## 3. 线程池

线程池是指管理一组同构工作线程的资源池，与工作线程联系紧密。工作线程从工作队列中取出任务，然后提交任务到线程池，然后继续等待下一个任务到达。

线程池有许多优点：

1. 重用线程，可以降低线程创建与销毁时的开销
2. 无需等待创建线程，所以提高了响应速度
3. 可以灵活控制线程数量，既不让处理器空闲，又不耗尽内存

### newFixedThreadPool

固定线程池的长度,每提交一个任务就创建一个线程，直到达线程池的最大长度，这时线程池的规模将不会再变化（如果某个线程由于发生了未预期的Exception而结束那么线程池将会补充一个线程）。 

### newCachedThreadPool

创建一个可缓存的线程池，如果线程池的当前规模超过了处理需求时，那么将会回收空闲的线程，而当需求增加时，则添加新的线程，线程的规模不存在 任何的限制。

### newSingleThreadExecutor

是一个单线程的Executor，它创建单个线程来执行任务，如果这个线程异常结束，会创建另外一个线程来替代。newSingleThreadExecutor能确保依照任务在队列中顺序来串行执行（如FIFO、FIFO、优先级）。

### newScheduledThreadPool

newScheduledThreadPool创建了一个固定长度的线程池，而且以延迟或者定时的方式来执行任务，类似Timer。



## 4. Executor 的生命周期

当用完一个线程池时，需要将线程池关闭，否则JVM将无法退出。

关闭线程池有两种方法：

- shutdown： 此方法是平缓的关闭方式，不再接受新的任务，等待以已经启动的任务结束，当所有的任务完成，线程池中的线程死亡。
- shutdownNow：暴力关闭方式，取消尚未开始的任务并试图中断正在运行的线程。

ExecutorService的生命周期有三种：运行、关闭、终止。Executor初始创建时处于运行状态，执行shutdown之后进入关闭状态，等所有任务都完成后进入终止状态。可以调用 awaitTermination 等待 ExecutorService 到达终止状态，或者使用 isTerminated 轮询状态。

使用线程池的一般逻辑：

1. 调用Executors类的静态方法newCachedThreadPool或者newFixedThreadPoo
2. 调用submit提交任务（Runnable或Callable对象）
3. 如果想要取消一个任务，或者提交Callable对象，要保存好返回的Future对象
4. 当不再提交新任务时，调用shutdown。







### 未完待续。。。。。。


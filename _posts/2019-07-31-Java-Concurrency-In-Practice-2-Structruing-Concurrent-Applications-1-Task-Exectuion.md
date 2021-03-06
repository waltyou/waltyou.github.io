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

```java
public interface ExecutorService {
    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination (long timeout, TimeUnit unit) throws InterruptedException;

    //.....一些其他用于任务提交的便利方法.
}
```

关闭线程池有两种方法：

- shutdown： 此方法是平缓的关闭方式，不再接受新的任务，等待以已经启动的任务结束，当所有的任务完成，线程池中的线程死亡。
- shutdownNow：暴力关闭方式，取消尚未开始的任务并试图中断正在运行的线程。

ExecutorService的生命周期有三种：运行、关闭、终止。Executor初始创建时处于运行状态，执行shutdown之后进入关闭状态，等所有任务都完成后进入终止状态。可以调用 awaitTermination 等待 ExecutorService 到达终止状态，或者使用 isTerminated 轮询状态。

使用线程池的一般逻辑：

1. 调用Executors类的静态方法newCachedThreadPool或者newFixedThreadPoo
2. 调用submit提交任务（Runnable或Callable对象）
3. 如果想要取消一个任务，或者提交Callable对象，要保存好返回的Future对象
4. 当不再提交新任务时，调用shutdown。



## 5. 延迟任务与周期任务

通过ScheduledThreadPoolExecutor来代替Timer,TimerTask。

- Timer基于绝对时间，ScheduledThreadPoolExecutor基于相对时间。
- Timer执行所有定时任务只能创建一个线程，若某个任务执行时间过长，容易破坏其他TimerTask的定时精确性。
- Timer不捕获异常，Timetask抛出未检查的异常会终止定时器线程，已经调度但未执行的TimerTask将不会再执行，新的任务也不会被调度，出现"线程泄漏"

以下是错误的例子：

```java
public class OutOfTime {
    public static void main(String[] args) throws InterruptedException {
        Timer timer = new Timer();
        timer.schedule(new ThrowTask(), 1); //第一个任务抛出异常
        Thread.sleep(1000); 
        timer.schedule(new ThrowTask(), 1); //第二个任务将不能再执行, 并抛出异常Timer already cancelled.
        Thread.sleep(5000);
        System.out.println("end.");
    }
    
    static class ThrowTask extends TimerTask{

        @Override
        public void run() {
            throw new RuntimeException("test timer's error behaviour");
        }
    }
}
```



# 找出可利用的并行性

## 1. 携带结果的任务Callable与Future

Runnable的缺陷：不能返回一个值，或抛出一个异常。

Callable和Runnable都描述抽象的计算任务，Callable可以返回一个值，并可以抛出一个异常。

```java
public interface Callable<V> {
	V call() throws Exception;
}
```

Future表示了一个任务的生命周期，提供了相应的方法判断是否完成或被取消以及获取执行结果：

```java
public interface Future<V> {
  boolean cancel(boolean mayInterruptIfRunning);
  boolean isCancelled();
  boolean isDone();
  V get() throws Exception;
  V get(long timeout, TimeUnit unit);
}
```

- get方法：若任务完成，返回结果或抛出ExecutionException；若任务取消，抛出CancellationException；若任务没完成，阻塞等待结果
- ExecutorService的submit方法提交一个Callable任务，并返回一个Future来判断执行状态并获取执行结果
- 安全发布过程：将任务从提交线程穿个执行线程，结果从计算线程到调用get方法的线程

异构任务并行化中存在的局限：

> 只有大量相互独立且同构的任务可以并发进行处理时，才能体现出性能的提升。

## 2. 为任务设定时限

如果超出期望执行时间，将不要其结果。

### Future.get

```java
public class RenderWithTimeBudget {
    private static final Ad DEFAULT_AD = new Ad();
    private static final long TIME_BUDGET = 1000;
    private static final ExecutorService exec = Executors.newCachedThreadPool();

    Page renderPageWithAd() throws InterruptedException {
        long endNanos = System.nanoTime() + TIME_BUDGET;
        Future<Ad> f = exec.submit(new FetchAdTask());
        // Render the page while waiting for the ad
        Page page = renderPageBody();
        Ad ad;
        try {
            // Only wait for the remaining time budget
            long timeLeft = endNanos - System.nanoTime();
            ad = f.get(timeLeft, NANOSECONDS);
        } catch (ExecutionException e) {
            ad = DEFAULT_AD;
        } catch (TimeoutException e) {
            ad = DEFAULT_AD;
            f.cancel(true);
        }
        page.setAd(ad);
        return page;
    }

    Page renderPageBody() { return new Page(); }


    static class Ad {
    }

    static class Page {
        public void setAd(Ad ad) { }
    }

    static class FetchAdTask implements Callable<Ad> {
        public Ad call() {
            return new Ad();
        }
    }

}
```

### invokeAll()

线程池支持通过invokeAll()可以一次批量提交多个callable。这个方法传入一个callable的集合，然后返回一个future的列表。该方法会阻塞直到所有callable执行完成。

### invokeAny()

与invokeAll相对应的，线程池还提供了一个invokeAny()方法，该方法将会阻塞直到第一个callable完成然后返回这一个callable的结果。




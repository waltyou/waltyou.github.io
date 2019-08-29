---
layout: post
title: Java 并发编程实战-学习日志（二）2：取消与关闭
date: 2019-08-15 17:15:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

Java没有提供任务安全结束线程的机制，提供了中断，这是一种协作机制：使一个线程终止另一个线程。

为什么是协作机制：1、立即停止会造成数据结构的不一致性 2、任务本身比其他线程更懂得如何清除当前正在执行的任务

软件质量的区别：良好的软件能很好的处理失败、关闭、结束等过程。

<!-- more -->

---

* 目录
{:toc}
---

# 任务取消


外部代码，能够将某个操作正常完成之前，将其置入完成状态，那么这个操作就称为**可取消的（Cancellable）**。

取消操作的原因有很多：
1. 用户请求取消。
2. 有时间限制的操作，如超时设定。
3. 应用程序事件。
4. 错误。
5. 关闭。

如下面这种取消操作实现：

```java
/**
 * 一个可取消的素数生成器
 * 使用volatile类型的域保存取消状态
 * 通过循环来检测任务是否取消
 */
@ThreadSafe
public class PrimeGenerator implements Runnable {
	private final List<BigInteger> primes = new ArrayList<>();
	private volatile boolean canceled;
	
	@Override
	public void run() {
		BigInteger p = BigInteger.ONE;
		while (!canceled){
			p = p.nextProbablePrime();
			synchronized (this) { //同步添加素数
				primes.add(p);
			}
		}
	}
	
	/**
	 * 取消生成素数
	 */
	public void cancel(){
		canceled = true;
	}
	
	/**
	 * 同步获取素数
	 * @return 已经生成的素数
	 */
	public synchronized List<BigInteger> get(){
		return new ArrayList<>(primes);
	}
}
```

其测试用例:

```java
public class PrimeGeneratorTest {
	public static void main(String[] args) {
		PrimeGenerator pg = new PrimeGenerator();
		new Thread(pg).start();
		
		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally{
			pg.cancel(); //始终取消
		}
		
		System.out.println("all primes: " + pg.get());
	}
}
```



## 1. 中断

- 调用`interrupt`并不意味者立即停止目标线程正在进行的工作，而只是传递了请求中断的消息。会在下一个取消点中断自己，如wait，sleep，join等。

- 通常，中断是实现取消的最合理方式。

```java
public class PrimeProducer extends Thread {
    private final BlockingQueue<BigInteger> queue;

    PrimeProducer(BlockingQueue<BigInteger> queue) {
        this.queue = queue;
    }

    public void run() {
        try {
            BigInteger p = BigInteger.ONE;
            while (!Thread.currentThread().isInterrupted())
                queue.put(p = p.nextProbablePrime());
        } catch (InterruptedException consumed) {
            /* Allow thread to exit */
        }
    }

    public void cancel() {
        interrupt();
    }
}
```

## 2. 中断策略

中断策略规定线程如何解释某个中断请求——当发现中断请求时，应该做哪些工作，哪些工作单元对于中断来说是原子操作，以及以多快的速度来响应中断。

最合理的中断策略是以某种形式的线程级取消操作或者服务级取消操作：尽快退出，必要时进行清理，通知某个所有者该线程已经退出。此外还可以建立其他的中断策略，例如暂停服务或重新开始服务。

区分任务和线程对中断的反应非常重要。一个中断请求可以有一个或者多个接受者——中断线程池中的某个工作者线程，同时意味着“取消当前任务”和“关闭工作者线程”。

线程应该只能由其所有者中断，所有者可以将线程的中断策略信息封装到某个合适的取消机制中，例如关闭方法。

> 由于每个线程拥有各自的中断策略，因此除非你知道中断对该线程的含义，否则就不应该中断这个线程。



## 3. 响应中断

在调用可中断的阻塞函数时，例如Thread.sleep或BolckingQueue.put等，有两种实用策略可以处理InterruptedException:

- 传递异常
- 恢复中断状态

将InterruptedException传递给调用者：

```java
BlockingQueue<Task> queue;
public Task getNextTask() throws InterruptedException{
    return queue.take();
}
```

如果不想或者无法传递InterruptedException(或许通过Runnable来定义任务)，那么需要寻找另一种方式来保存中断请求。一种标准的方法就是通过再次调用interrupt来恢复中断状态。

> 只有实现了线程中断策略的代码才可以屏蔽中断请求，在常规的任务和库代码中都不应该屏蔽中断请求。

对于不支持取消但仍可以调用可中断阻塞方法的操作，他们必须在循环中调用这些方法，并在发现中断后重新尝试。在这种情况下，他们应该在本地保存中断状态，并在返回前回复状态而不是在捕获InterruptedException时恢复状态。如果过早的设置中断状态，就可能引起无限循环，因为大多数可中断的阻塞方法都会在入口处检查中断状态，并且当发现该状态已被设置时会立即抛出InterruptedException(通常，可中断的方法会在阻塞或进行重要的工作前首先检查中断，从而尽快的响应中断)。

## 4.  示例： 计时运行

```java
private static final ScheduledExecutorService cancelExec = ...;
public static void timedRun(Runnable r,long timeout,TimeUnit unit) {
    final Thread taskThread = Thread.currentThread();
    cancelExec.schedule(new Runnable(){
        public void run(){
            taskThread.interrupt();
        }
    },timeout,unit);
    r.run();
}
```

这是一种非常简单的方法，然而却破坏了以下规则：**在中断线程之前，应该了解它的中断策略**。由于timedRun可以从任意一个线程中调用，因此它无法知道这个调用线程的中断策略。如果任务在超时之前完成，那么中断timedRun所在线程的取消任务将在timedRun返回到调用者后启动。

而且，如果任务不响应中断，那么timedRun会在任务结束时才返回，此时可能已经超过了指定的时限。

```java
public static void timedRun(final Runnable r,long timeout,TimeUnit unit) 
  throws InterruptedException{
    class RethrowableTask implements Runnable{
        private volatile Throwable t;
        public void run(){
            try{
                r.run();
            } catch(Throwable t){
                this.t = t;
            }
        }
        void rethrow(){
            if(t != null) {
                throw launderThrowable(t);
            }
        }    
    }
    
    RethrowableTask task = new RethrowableTask();
    final Thread taskThread = new Thread(task);
    taskThread.start();
    cancelExec.schedule(new Runnable(){
          public void run() {
              taskThread.interrupt();
          }
      },timeout,unit);
    taskThread.join(unit.toMillis(timeout));
    task.rethrow();
}
```



## 4. 通过 Future 来实现取消

`ExecutorService.submit` 返回一个Future来描述任务。

Future有一个cancel方法，该方法带有一个boolean类型的参数`mayInterruptIfRunning`，表示取消操作是否成功(这只是表示任务是否能够接收中断，而不是表示任务能否检测并处理中断)。如果该参数为true并且任务当前正在某个线程中运行，那么这个线程能被中断。如果这个参数为false，那么意味着“若任务还没有启动，那就不要启动它”，这种方式应该用于那些不处理中断的任务中。

除非你清楚线程的中断策略，否则不要中断线程。当尝试取消某个任务时，不宜直接中断线程池。

```java
public static void timeRun(Runnable r,long timeout,TimeUnit unit) throws InterruptedException{
    Future<?> task = taskExec.submit(r);
    try{
        task.get(timeout,unit);
    } catch(TimeoutException e){
        //接下来任务将被取消
    } catch(ExecutionException e){
        throw launderThrowable(e.getCauese());
    } finally {
        //如果任务已经结束，那么执行取消操作也不会带来任何影响
        //如果任务正在运行，那么将被打断
        task.cancel(true);
    }
}
```

> 当Future.get抛出InterruptedException或TimeoutException 时，如果你知道不再需要结果，那么就可以调用Future.cancel取消任务。



## 5. 处理不可中断的阻塞

并非所有的可阻塞方法或者阻塞机制都能响应中断；如果一个线程由于执行同步的Socket I/O或者等待获得内置锁而阻塞，那么中断请求只能设置线程的中断状态，除此之外没有其他任何作用。

由于执行不可中断操作而被阻塞的线程，可以使用类似于中断的手段来停止这些线程，但这要求我们必须知道线程阻塞的原因。

1. java.io包中的同步Socket I/O。在服务器应用程序中，最常见的阻塞I/O形式就是对套接字进行读取和写入。虽然InputStream和OutputStream中的read和write等方法都不会响应中断，但是通过关闭底层的套接字，可以使得由于执行read或write等方法被阻塞的线程抛出一个SocketException
2. java.io包中的同步I/O。当中断一个正在InterruptibleChannel上等待的线程时，将抛出ClosedByInterruptException并关闭链路。当关闭一个InterruptibleChannel时，将导致所有在链路操作上阻塞的线程都抛出AsynchronousCloseException。大多数的Channel都实现了InterruptibleChannel.
3. Selector的异步I/O。如果一个线程在调用Selector.select方法时阻塞了，那么调用close或wakeup方法会使线程抛出ClosedSelectorException并提前返回。
4. 获取某个锁。如果一个线程由于等待某个内置锁而阻塞，那么将无法响应中断。在Lock类中提供了lockInterruptibly方法，该方法允许在等待一个锁的同时仍能响应中断。

下面展示的是如何封装非标准的取消操作。

```java
public class ReaderThread extends Thread{
    private final Socket socket;
    private final InputStream in;
    public ReaderThread(Socket socket) throws IOException{
        this.socket = socket;
        this.in = socket.getInputStream();
    }
    
    public void interrupt(){
        try{
            socket.close();
        }catch(IOException ignored){
            
        }finally{
            super.interrupt();
        }
        
    }
    
    
    public void run(){
        try{
            byte[] buf = new byte[1024];
            while(true){
                int count = in.read(buf);
                if(count < 0) {
                    break;
                } else if(count > 0) {
                    processBuffer(buf,count);
                }
            }
        } catch(IOException e){
            /*允许线程退出*/
        }
    }
}

```






## 未完待续。。。。。。





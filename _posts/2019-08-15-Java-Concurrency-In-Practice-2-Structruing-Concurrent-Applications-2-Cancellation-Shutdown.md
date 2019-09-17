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



## 6. 采用newTaskFor封装非标准的取消

newTaskFor是一个工厂方法，它将创建Future来代表任务。newTaskFor还能返回一个RunnableFuture接口，该接口拓展了Future和Runnable(并由FutureTask实现)。

当把一个Callable提交给ExecutorService时，submit方法会返回一个Future，我们可以通过这个Future来取消任务。

通过定制表示任务的Future可以改变Future.cancel的行为。例如定制的取消代码可以实现日志记录或者收集取消操作的统计信息，以及取消一些不响应中断的操作。通过改写interrupt方法，ReaderThread可以取消基于套接字的线程。同样，通过改写任务的Future，cancel方法也可以实现类似的功能。

```java
public interface CancellableTask<T> extends Callable<T>{
    void cancel();
    RunnableFuture<T> newTask();
}

public class CancellingExecutor extends ThreadPoolExecutor{
    protected<T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        if(callable instanceof CancellableTask){
            return ((CancellableTask<T>)callable).newTask();
        } else {
            return super.newTaskFor(callable);
        }
    }
}

public abstract class SocketUsingTask<T> implements CancellableTask<T> {
    private Socket socket;
    protected synchronized void setSocket(Socket s){
        socket = s;
    }
    
    public synchronized void cancel(){
        try{
            if(socket!=null){
                socket.close();
            } 
        } catch(IOException ingnored){}
    }
    
    public RunnableFuture<T> newTask(){
        return new FutureTask<T>(this){
            public boolean cancel(boolean mayInterruptIfRunning){
                try{
                    SocketUsingTask.this.cancel();
                } finally {
                    return super.cancel(mayInterruptIfRunning);
                }
            }
        };
    }
}
```

---

# 停止基于线程的服务

应用程序通常会创建拥有多个线程的服务，例如线程池，并且这些服务的生命周期通常比创建它们的生命周期更长。如果应用程序准备退出，那么这些服务所拥有的线程也需要结束。由于无法通过抢占式的方法来停止线程，因此它们需要自行结束。

正确的封装原则是：除非拥有某个线程，否则不能对该线程进行操控，例如，中断线程或者修改线程优先级等。

> 对于持有线程的服务，主要服务的存在时间大于创建线程的方法的存在时间，那么就应该提供生命周期方法。

## 1. 示例：日志服务

```java
public class LogWriter{
    private final BlockingQueue<String> queue;
    private final LoggerThread thread;
    
    public LogWriter(Writer writer){
        this.queue = new BlokingQueue<>();
        this.logger = new LoggerThread(write);
    }
    
    public void start(){
        logget.start();
    }
    
    public void log(String msg) throws InterruptedException{
        queue.put(msg);
    }
    
    private class LoggerThread extends Thread{
        private final PrintWriter writer;;
        
        public void run(){
            try{
                while(true){
                    writer.println(queue.take());
                }
            } catch(InterruptedException ignored){
                
            } finally {
                writer.close();
            }
            
        }
    }
}
```

要停止日志线程是很容易的，因为它会反复调用take，而take能响应中断。如果日志线程修改为捕获到InterruptedException时退出，那么只需要中断日志线程就能停止服务。

然而，如果只是使日志线程退出，那么还不是一种完备的关闭机制。这种直接关闭的做法会丢失那些正在等待被写入到日志的信息，不仅如此，其他线程将在调用log时被阻塞，因为日志消息队列是满的，因此这些线程将无法解除阻塞状态。

另一种关闭LogWriter的方法是：设置某个“已请求关闭”标志，避免进一步提交日志消息。

```java
 public void log(String msg) throws InterruptedException{
    if(!shutdownRequested){
        queue.put(msg);
    }else {
        throw new IllegalStateException("logger is shut down");
    }
}
```

为LogWriter提供可靠关闭操作的方法是解决竞态条件问题，因而要使日志消息的提交操作成为原子操作。

```java
public class LogService{
    private final BlockingQueue<String> queue;
    private final LoggerThread loggerThread;
    private final PrintWriter writer;
    private boolean isShutdown;
    private int reservations;
    
    public void start(){
        loggerThread.start();
    }
    
    public void stop(){
        synchronized(this){
            isShutdown = true;
        }
        loggerThread.interrupt();
    }
    
    public void log(String msg) throws InterruptedException{
        synchronized(this){
            if(isShutdown){
                throw new IllegalStateException(...);
            }
            ++reservations;
        }
        queue.put(msg);
    }
    
    private class LoggerThread extends Thread{
        public void run(){
            try{
                while(true){
                    try{
                        synchronized(LogService.this){
                            if(isShutdown && reservations == 0){
                                break;
                            }
                            String msg = queue.take();
                            synchronized(LogService.this){
                                --reservations;
                            }
                            writer.println(msg);                        
                        } catch(InterruptedException e){
                            /* retry */
                        }
                    } 

                }
            } finally {
                writer.close();
            }
        }
    }
}
```

## 2. 关闭ExecutorService

在复杂程序中，通常会将ExecutorService封装在某个更高级别的服务中，并且该服务能提供其自己的生命周期方法。

```java
public class LogService{
    private final ExecutorService exec = newSingleThreadExecutor();
    
    public void start();
    
    public void stop() throws InterruptedException {
        try{
            exec.shutdown();
            exec.awaitTermination(TIMEOUT,UNIT);
        } finally {
            writer.close();
        }
    }
    
    public void log(String msg) {
        try{
            exec.execute(new WriteTask(msg));
        } catch (RejectedExecutionException ignored) {
            
        }
    }
}
```

## 3. “毒丸” 对象

另一种关闭生产者-消费者服务的方式，就是使用“毒丸”对象。毒丸是指一个放在队列上的对象，其含义是：当得到足够对象时，立即停止。在FIFO队列中，“毒丸”对象确保消费者在关闭之前首先完成队列的所有工作，在提交毒丸对象之前提交的工作都会被处理，而生产者在提交了毒丸对象后，将不会提交任何工作。

```java
public class IndexingService{
    private static final File POISON = new File("");
    private final IndexerThread consumer = new IndexerThread();
    private final CrawlerThread producer = new CrawlerThread();
    private final BlockingQueue<File> queue;
    private final File root;
    
    public void start(){
        producer.start();
        consumer.start();
    }
    
    public void stop(){
        producer.interrupt();
    }
    
    public void awaitTermination() throws InterruptedException{
        consumer.join();
    }
    
    class CrawlerThread extends Thread{
        public void run(){
            try{
                crawl(root);
            } catch(InterruptedException e){
                
            } finally {
                while(true) {
                    try{
                        queue.put(POISON);
                        break;
                    } catch(InterruptedException e1){
                        /*重新尝试*/
                    }
                }
            }
        }
        
        private void crawl(File root) throws InterruptedException{
            ...
        }
    }
    
    class IndexerThread extends Thread{
        public void run(){
            try{
                while(true){
                    File file = queue.take();
                    if(file == POISON){
                        break;
                    } else {
                        indexFile(file);
                    }
                }
            } catch(InterruptedException consumed){
                
            }
        }
    }
}
```

## 4. 示例：只执行一次的服务

如果某个方法需要处理一批任务，并且当所有任务都处理完成后才返回，那么可以通过一个私有的Executor来简化服务的生命周期管理，其中该Executor的生命周期是由这个方法来控制的(这种情况下，invokeAll和invokeAny等方法通常会起较大的作用).

```java
boolean checkMail(Set<String> hosts,long timeout,TimeUnit unit) throws InterruptedException{
    ExecutorService exec = Executors.newCachedThreadPool();
    final AtomicBoolean hasNewMail = new AtomicBoolean(false);
    
    try{
        for(final String host : hosts){
            exec.execute(new Runnable(){
                public void run(){
                    if(checkMail(host)){
                        hasNewMail.set(true);
                    }
                }
            });
        }
    } finally {
        exec.shutdown();
        exec.awaitTermination(timeout,unit);
    }
    return hasNewMail.get();
}
```

checkMail能在多台主机上并行的检查新邮件。它创建了一个私有的Executor,并向每台主机提交一个任务。然后，当所有邮件检查任务都执行完成后，关闭Executor并等待结束。之所以采用AtomicBoolean是因为能从内部的Runnable中访问hasNewMail标识。

## 5. shutdownNow的局限性

当通过shutdownNow来强行关闭ExecutorService时，它会尝试取消正在运行的任务并返回所有已提交但尚未开始的任务，从而将这些任务写入日志或者保存起来以便之后进行处理。

我们无法通过常规方法来找出哪些任务已经开始但尚未结束。这意味着我们无法在关闭过程中知道正在执行的任务的状态，除非任务本身会执行某种检查。

TrackingExecutor中给出了如何在关闭过程中判断正在执行的任务。

```java
public class TrackingExecutor extends AbstractExecutorService{
    private final ExecutorService exec;
    private final Set<Runnable> tasksCancelledAtShutdown = Collections.synchronizedSet(new HashSet<Runnable>());
    
    public List<Runnable> getCancelledTasks(){
        if(!exec.isTerminated){
            throw new IllegalStateException(..);
        }
        return new ArrayList<Runnable>(tasksCancelledAtShutdown);
    }
    
    public void execute(final Runnable runnable){
        exec.execute(new Runnable(){
            public void run(){
                try{
                    runnable.run();
                } finally {
                    if(isShutdown() && Thread.currentThread().isInterrupted()){
                        tasksCancelledAtShutdown.add(runnable);
                    }
                }
            }
        });
    }
}
```

使用TrackingExecutorService来保存未完成的任务以备后续执行.

```java
public abstract class WebCrawler{
    private volatile TrackingExecutor exec;
    private final Set<URL> urlsToCrawl = new HashSet<URL>();
    
    public synchronized void start(){
        exec = new TrackingExecutor(Executors.newCachedThreadPool());
        for(URL url : urlsToCrawl) {
            submitCrawlTask(url);
        }
        urlsToCrawl.clear();
    }
    
    public synchronzied void stop() throws InterruptedException {
        try{
            saveUncrawled(exec.shutdownNow());
            if(exec.awaitTermination(TIMEOUT,UNIT)){
                saveUncrawled(exec.getCancelledTasks());
            }
        } finally {
            exec = null;
        }
    }
    
    protected abstract List<URL> processPage(URL url);
    
    private void saveUncrawled(List<Runnable uncrawled){
        for(Runnable task : uncrawled){
            urlsToCrawl.add(((CrawlTask) task).getPage());
        }
    }
    
    private void submitCrawlTask(URL url){
        exec.execute(new CrawlTask(u));
    }
    
    private class CrawlTask implements Runnable{
        private final URL url;
        
        public void run(){
            for(URL link : processPage(url)){
                if(Thread.currentThread().isInterrupted()){
                    return;
                }
                submitCrawlTask(link);
            }
        }
        
        public URL getPage(){
            return url;
        }
    }
    
}
```

在TrackingExecutor中存在一个不可避免的竞态条件，从而产生“误报”问题：一些被认为已经取消的任务实际上已经执行完成。原因在于，在任务执行最后一条指令以及线程池将任务记录为结束的两个时刻之间，线程池可能被关闭。如果任务是幂等的(Idempotent,即将任务执行两次和执行一次会得到相同的结果)，那么不会存在问题。

---

# 处理非正常的线程终止

当并发程序中的某个线程发生故障使控制台中可能会输出栈追踪信息，但是没有人会观察控制台。此外，当线程发生故障时，应用程序可能看起来仍然在工作，所以这个失败很可能被忽略。幸运的是，我们有可以监测并防止程序中“遗漏”线程的方法。

导致线程死亡的最主要原因就是RuntimeException。这是unchecked异常，程序默认会在控制台输出栈追踪信息，并终止线程。

典型的线程池工作者线程结构:

```java
public void run(){
    Throwable thrown = null;
    try{
        while(!isInterrupted()){
            runTask(getTaskFromWorkueue());
        }
    } catch (Throwable e) {
        thrown = e;
    } finally {
        threadExited(this,throw);
    }
}
```

Thread api中同样提供了UncaughtExceptionHandler,它能检测出某个线程由于未捕获的异常而终结的情况。

当一个线程由于未捕获异常而退出时，JVM会把这个事件报告给应用程序提供的UncaughtExceptionHandler异常处理器。如果没有提供任何异常处理器，那么默认的行为是将栈追踪信息输出到System.err.

```java
public interface UncaughtExceptionHandler {
    void uncaughtException(Thread t,Throwable e);
}
```

异常处理器如何处理未捕获异常，取决于对服务质量的需求。最常见的响应方式是将一个错误信息以及相应的栈追踪信息写入应用程序日志中。

```java
public class USHLogger implements Thread.UncaughtExceptionHandler {
    public void uncaughtException(Thread t,Throwable e){
        Logger logger = logger.getAnonymousLogger();
        logger.log(Level.SEVERE,"Thread terminated with exception :" + t.getName()),e);
    }
}
```

> 在运行时间较长的应用程序中，通常会为所有线程的未捕获异常指定同一个异常处理器，并且该处理器至少会将异常信息记录到日志中。

要为线程池中的所有线程设置一个UncaughtExceptionHandler,需要为ThreadPoolExecutor的构造函数提供一个ThreadFactory。标准线程池允许当发生未捕获异常时结束线程，但由于使用了一个try-finally代码来接收通知，因此当线程结束时，将有新的线程来代替它。如果没有提供捕获异常处理器或者其他的故障通知机制，那么任务会悄悄失败，从而导致极大的混乱。如果你希望在任务由于发送异常而失败时获得通知并且执行一些特定于任务的恢复操作，那么可以将任务封装在能捕获异常的Runnable或Callable中，或者改写ThreadPoolExecutor的afterExecute方法。

令人困惑的是，只有通过execute提交的任务，才能将它抛出的异常交给未捕获异常处理器，而通过submit提交的任务，会被封装成ExecutionException抛出。

---

# JVM关闭

JVM既可以正常关闭也可以强行关闭。

正常关闭的触发方式有多种，包括：当最后一个“非守护“线程结束时，或者调用System.exit时，或者通过其他特定平台的方法关闭时(例如发送了SIGINT信号或键入crtl + C)。但也可以调用Runtime.halt或者在操作系统中杀死JVM进程来强行关闭JVM.

## 1. 关闭钩子

在正常关闭中，JVM首先调用所有已注册的关闭钩子(Shutdown hook)。关闭钩子是指通过Runtime.addShutdownHook注册的但尚未开始的线程。JVM并不能保证关闭钩子的调用顺序。在关闭应用程序线程中，如果线程仍然在运行，那么这些线程接下来和关闭进程并发执行。如果runFinalizerOnExit为true。那么JVM将运行终结器，然后再停止。JVM并不会停止或中断任何在关闭时仍然运行的应用程序线程。当JVM最终结束时，这些线程将被强行结束。如果关闭钩子或终结器没有执行完成，那么正常关闭进程“挂起”并且JVM必须被强行关闭。当强行关闭时，只是关闭JVM，而不会运行关闭钩子。

关闭钩子应该是线程安全的：它们在访问共享数据时必须使用同步机制，并且小心的避免死锁，这和其他并发代码的要求相同。而且，关闭钩子不应该对应用程序的状态或者JVM的关闭原因作出任何假设。最后，关闭钩子应该尽快退出，因为它们会延迟JVM的结束时间，而用户可能希望JVM尽快终止。

关闭钩子可以用于实现服务或应用程序的清理工作，例如清理临时文件。

由于关闭钩子将并发执行，因此在关闭日志文件时可能导致其他需要日志服务的关闭钩子产生问题。实现这种功能的一种方式是对所有服务使用同一个关闭钩子，并且在关闭钩子中执行一系列的关闭操作。

```java
public void start(){
    Runtime.getRuntime().addShutdownHook(new Thread(){
        public void run(){
            try{
                LogService.this.stop();
            } catch(InterruptedException ignored){
                
            }
        }
    }
}
```






## 未完待续。。。。。。





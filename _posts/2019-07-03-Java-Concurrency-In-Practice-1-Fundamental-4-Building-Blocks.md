---
layout: post
title: Java 并发编程实战-学习日志（一）4：基础构建模块
date: 2019-07-03 21:13:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

这一章主要介绍 Java 中有用的并发模块。

<!-- more -->

* 目录
{:toc}
---

# 同步容器类

常见的有 Vector 和 Hashtable，还有一些通过 Collections.synchronizedXxx 等工厂方法创建的同步容器，它们实现同步的方式都是：将状态封装起来，并对每个公有方法都进行同步，使得每次只有一个线程能访问容器的状态。

## 1. 同步容器类的问题

同步容器都是线程安全的，但在某些情况下可能需要额外的客户端加锁操作来保护符合操作。在容器上常见的符合操作包括：迭代、跳转以及“先检查再执行”等。

### 先检查再执行

```java
public static Object getLast(Vector list){
    int lastIndex = list.size()-1;
    return list.get(lastIndex);
}

public static Object deleteLast(Vector list){
    int lastIndex = list.size()-1;
    list.remove(lastIndex);
}
```

### 迭代

```java
for (int i = 0; i < vector.size(); i++)
  doSomething(vector.get(i));
```

上面的代码在多线程情况下，可能会抛出 ArrayIndexOutOfBoundsException。



## 2. 迭代器与 ConcurrentModificationException

对容器进行迭代的标准方式是使用迭代器Iterator。然而在设计同步容器的迭代器时并没有考虑并发修改问题，如果在对同步容器进行迭代的过程中有线程修改了容器，那么就会失败。并且它们的表现行为是“及时失败”的。这意味着当它们发现容器在迭代过程中被修改时，就会抛出ConcurrentMondificationException异常。

```java
List<Widget> widgetList = Collections.synchronizedList(new ArrayList<Widget>());
...
// May throw ConcurrentModificationException
for (Widget w : widgetList)
  doSomething(w);
```

## 3. 隐藏迭代器

```java
public class HiddenIterator {

  private final Set<Integer> set = new HashSet<Integer>();
  public synchronized void add(Integer i) { set.add(i); } 
  public synchronized void remove(Integer i) { set.remove(i); }
  
  public void addTenThings() { 
    Random r = new Random();
    for (int i = 0; i < 10; i++)
	add(r.nextInt());
    System.out.println("DEBUG: added ten elements to " + set);
  }
}
```

上面代码在 System.out.println 时就可能产生异常，因为 set 的 toString 方法会对set 进行迭代。

> 正如封装对象的状态有助于维持不变性条件一样，封装对象的同步机制同样有助于确保实施同步策略。



---



# 并发容器

> 通过并发容器来替代同步容器，可以极大地提高伸缩性并降低风险。

## 1. ConcurrentHashMap

在 Java 5 中增加了 ConcurrentHashMap，它是线程安全的，但是它不像 HashTable 或者 SynchronizeMap 一样，所有方法公用一个锁，而是采用了**分段锁**。这样子不仅在多线程环境下极大地提高了吞吐量，而在单线程下只会损失一小部分性能。



## 2. 额外的原子 Map 操作

由于无法对  ConcurrentHashMap 不能被加锁来执行独占访问，因此无法通过客户端加锁来新增一些原子操作。倒是一些常见的复合操作，例如“若没有则添加”、“若相等则移除”等，已经在 ConcurrentMap 接口中定义好了，如下：

```java
public interface ConcurrentMap<K,V> extends Map<K,V> {
  // Insert into map only if no value is mapped from K
  V putIfAbsent(K key, V value);
  // Remove only if K is mapped to V
  boolean remove(K key, V value);
  // Replace value only if K is mapped to oldValue
  boolean replace(K key, V oldValue, V newValue);
  // Replace value only if K is mapped to some value
  V replace(K key, V newValue); 
}
```



## 3. CopyOnWriteArrayList

CopyOnWriteArrayList 用于替代同步list。

“写入时复制”容器的线程安全性在于，只要正确地发布了一个事实不可变对象，那么在访问该对象时就不需要进一步同步。在每次修改时，都会创建并重新发布一个新的容器副本，从而实现可变性。显然每当修改容器时都会复制底层数组。仅当**迭代操作远远多于修改操作**时，才应该使用"写入时复制"容器。



# 阻塞队列和生产者 - 消费者模式

**阻塞队列提供了可阻塞的put 和take方法， 以及支持定时的offer和poll方法**。如果队列已经满了， 那么put方法将阻塞直到有空间可用；如果队列为空， 那么take方法将会阻塞直到有元素可用。队列可以是有界的也可以是无界的， 无界队列永远都不会充满， 因此无界队列上的put方法也永远不会阻塞。

**阻塞队列支持生产者－消费者这种设计模式**。该模式将“找出需要完成的工作” 与“执行工作” 这两个过程分离开来， 并把工作项放入一个“ 待完成” 列表中以便在随后处理， 而不是找出后立即处理。生产者－ 消费者模式能简化开发过程， 因为它消除了生产者类和消费者类之间的代码依赖性， 此外， 该模式还将生产数据的过程与使用数据的过程解耦开来以简化工作负载的管理， 因为这两个过程在处理数据的速率上有听不同。

基于阻塞队列构建的生产者－消费者设计中， 当数据生成时， 生产者把数据放人队列，而当消费者准备处理数据时， 将从队列中获取数据。生产者不需要知道消费者的标识或数量，或者它们是否是唯一的生产者， 而只需将数据放入队列即可。同样， 消费者也不需要知道生产者是谁， 或者工作来自何处。BlockingQueue 简化了生产者－ 消费者设计的实现过程， 它支持任意数批的生产者和消费者。一种最常见的生产者－ 消费者设计模式就是线程池与工作队列的组合， 在Executor任务执行框架中就体现了这种模式。

阻塞队列简化了消费者程序的编码， 因为 take操作会一直阻塞直到有可用的数据。如果 生产者不能尽快地产生工作项使消费者保持忙碌， 那么消费者就只能一直等待， 直到有工作可做。在某些情况下， 这种方式是非常合适的（例如， 在服务器应用程序中，没有任何客户请求 服务）， 而在其他一些情况下， 这也表示需要调整生产者线程数量和消费者线程数量之间的比率，从而实现更高的资源利用率（例如，在 “ 网页爬虫[Web Crawler]"或其他应用程序中，有无穷的工作需要完成）。

如果生产者生成工作的速率比消费者处理工作的速率快， 那么工作项会在队列中累积起 ，最终耗尽内存。 同样， put方法的阻塞特性也极大地简化了生产者的编码。 如果使用有界队列， 那么当队列充满时， 生产者将阻塞并且不能继续生成工作， 而消费者就有时间来赶上工作处理进度。

阻塞队列同样提供了一个 offer方法，如果数据项不能被添加到队列中， 那么将返回一个失败状态。 这样你就能够创建更多灵活的策略来处理负荷过载的情况，例如减轻负载， 将多余 的工作项序列化并写入磁盘，减少生产者线程的数量，或者通过某种方式来抑制生产者线程。

> 在构建高可靠的应用程序时，有界队列是一种强大的资源管理工具：它们能抑制并防止产生过多的工作项，使应用程序在负荷过载的情况下变得更加健壮。

在类库中包含了BlockingQueue的多种实现，其中，LinkedBlockingQueue和ArrayBlocking­Queue是FIFO队列，二者分别与LinkedList和ArrayList类似，但比同步List拥有更好的并发性能。**PriorityBlockingQueue是一个按优先级排序的队列**，当你希望按照某种顺序而不是FIFO来处理元素时，这个队列将非常有用。正如其他有序的容器一样，PriorityBlockingQueue既可以根据元素的自然顺序来比较元素（如果它们实现了Comparable方法），也可以使用Comparator来比较。

最后一个BlockingQueue实现是SynchronousQueue, 实际上它不是一个真正的队列， 因为它不会为队列中元素维护存储空间。与其他队列不同的是， 它维护一组线程， 这些线程在等待着把元素加入或移出队列。如果以洗盘子的比喻为例， 那么这就相当千没有盘架， 而是将洗好的盘子直接放入下一个空闲的烘干机中。这种实现队列的方式看似很奇怪， 但由于可以直接交付工作，从而降低了将数据从生产者移动到消费者的延迟。（在传统的队列中， 在一个工作单元可以交付之前， 必须通过串行方式首先完成入列[Enqueue]或者出列[Dequeue]等操作。） 直接交付方式还会将更多关于任务状态的信息反馈给生产者。当交付被接受时，它就知道消费者已经得到了任务， 而不是简单地把任务放入一个队列一一这种区别就好比将文件直接交给同事，还是将文件放到她的邮箱中并希望她能尽快拿到文件。因为SynchronousQueue 没有存储功能，因此put 和take 会一直阻塞，直到有另一个线程已经准备好参与到交付过程中。仅当有足够多的消费者，并且总是有一个消费者准备好获取交付的工作时， 才适合使用同步队列。

## 1. 串行线程封闭

在java.util.coricurrent中实现的各种阻塞队列都包含了足够的内部同步机制， 从而安全地将对象从生产者线程发布到消费者线程。

对于可变对象， 生产者 － 消费者这种设计与阻塞队列一起， 促进了串行线程封闭， 从而将对象所有权从生产者交付给消费者。 **线程封闭对象只能由单个线程拥有， 但可以通过安全地发布该对象来 “转移 ” 所有权。**在转移所有权后， 也只有另一个线程能获得这个对象的访问权限，并且发布对象的线程不会再访问它。这种安全的发布确保了对象状态对于新的所有者来说是可见的， 并且由于最初的所有者不会再访问它， 因此对象将被封闭在新的线程中。 新的所有者线程可以对该对象做任意修改， **因为它具有独占的访问权**。

对象池利用了串行线程封闭， 将对象 “借给“一个请求线程。 只要对象池包含足够的内部同步来安全地发布池中的对象， 并且只要客户代码本身不会发布池中的对象， 或者在将对象返回给对象池后就不再使用它， 那么就可以安全地在线程之间传递所有权。

我们也可以使用其他发布机制来传递可变对象的所有权， 但必须确保只有一个线程能接受被转移的对象。 阻塞队列简化了这项工作。 除此之外， 还可以通过 ConcurrentMap 的原子方法remove 或者 AtomicReference 的原子方法 compareAndSet 来完成这项工作。

## 2. 双端队列与工作密取

Java 6 增加了两种容器类型， Deque (发音为 "deck") 和 BIockingDeque, 它们分别对 Queue 和 BlockingQueue 进行了扩展。 **Deque 是一个双端队列， 实现了在队列头和队列尾的高效插入和移除。 具体实现包括 ArrayDeque 和 LinkedBlockingDeque。**

正如阻塞队列适用于生产者 － 消费者模式， 双端队列同样适用于另一种相关模式， 即工作密取 (Work Stealing)。 在生产者－消费者设计中，所有消费者有一个共享的工作队列， 而在 工作密取设计中， 每个消费者都有各自的双端队列。 如果一个消费者完成了自己双端队列中的 全部工作， **那么它可以从其他消费者双端队列末尾秘密地获取工作**。 密取工作模式比传统的生产者－消费者模式具有更高的可伸缩性， 这是因为工作者线程不会在单个共享的任务队列上发 生竞争。 在大多数时候， 它们都只是访问自己的双端队列， 从而极大地减少了竞争。 当工作者线程需要访问另一个队列时， 它会从队列的尾部而不是从头部获取工作， 因此进一步降低了队列上的竞争程度。

工作密取非常适用于既是消费者也是生产者问题——当执行某个工作时可能导致出现更多的工作。 例如， 在网页爬虫程序中处理一个页面时， 通常会发现有更多的页面需要处理。 类似的还有许多搜索图的算法， 例如在垃圾回收阶段对堆进行标记， 都可以通过工作密取机制来实 现高效并行。 当一个工作线程找到新的任务单元时， 它会将其放到自己队列的末尾（或者在工作共享设计模式中， 放入其他工作者线程的队列中）。 当双端队列为空时， 它会在另一个线程的队列队尾查找新的任务， 从而确保每个线程都保持忙碌状态。



# 阻塞方法与中断方法

线程可能会阻塞或暂停执行， 原因有多种：等待I/O操作结束， 等待获得一个锁， 等待从Thread.sleep 方法中醒来， 或是等待另一个线程的计算结果。 **当线程阻塞时， 它通常被挂起，并处于某种阻塞状态** (BLOCKED、 WAITING或 TIMED_WAITING)。阻塞操作与执行时间很长的普通操作的差别在于，**被阻塞的线程必须等待某个不受它控制的事件发生后才能继续执行**， 例如等待 I/0 操作完成， 等待某个锁变成可用， 或者等待外部计算的结束。 **当某个外部事件发生时， 线程被置回 RUNNABLE状态，并可以再次被调度执行。**

当某方法抛出 Interrupted-Exception 时， 表示该方法是一个阻塞方法， 如果这个方法被中断， 那么它将努力提前结束阻塞状态。

**Thread提供了interrupt方法，用于中断线程或者查询线程是否已经被中断。**每个线程都有一个布尔类型的属性， 表示线程的中断状态， 当中断线程时将设置这个状态。

中断是一种协作机制。一个线程不能强制其他线程停止正在执行的操作而去执行其他的操作。

当在代码中调用了一个将抛出InterruptedException 异常的方法时， 你自己的方法也就变成了一个阻塞方法， 并且必须要处理对中断的响应。对于库代码来说， 有两种基本选择：

1. 传递lnterruptedException。避开这个异常通常是最明智的策略——只需把InterruptedException 传递给方法的调用者。传递lnterruptedException 的方法包括， 根本不捕获该异常， 或者捕获该异常， 然后在执行某种简单的清理工作后再次抛出这个异常。
2. 恢复中断。有时候不能抛出lnterruptedException, 例如当代码是Runnable 的一部分时。在这些情况下， 必须捕获InterruptedException, 并通过调用当前线程上的interrupt 方法恢复中断状态， 这样在调用栈中更高层的代码将看到引发了一个中断。



# 同步工具类

同步工具类可以是任何一个对象，只要它根据其自身的状态来协调线程的控制流。

阻塞队列可以作为同步工具类，其他类型的同步工具类还包括信号量(Semaphore)、 栅栏(Barrier) 以及闭锁(Latch)。

## 1. 闭锁

闭锁是一种同步工具类， 可以延迟线程的进度直到其到达终止状态。 闭锁的作用相当于一扇门：在闭锁到达结束状态之前， 这扇门 一直是关闭的，并且没有任何线程能通过，当到达结束状态时，这扇门会打开并允许所有的线程通过。 当闭锁到达结束状态后 将不会再 改变状态，因此这扇门将永远保持打开状态。

闭锁可以用来确保某些活动直到其他活动都完成后才继续执行，例如：

- 确保某个计算在其需要的所有资源都被初始化之后才继续执行
- 确保某个服务在其依赖的所有其他服务都已经启动之后才启动
- 等待直到某个操作的所有参与者（例如，在多玩家游戏中的所有玩家）都就绪再继续执行

CountDownLatch是一种灵活的闭锁实现，可以在上述各种情况中使用， 它可以使一个或多个线程等待一组事件发生。闭锁状态包括一个计数器，该计数器被初始化为一个正数， 表示需要等待的事件数量。countDown方法递减计数器， 表示有一个事件已经发生了，而 await 方 法等待计数器达到零，这表示所有需要等待的事件都已经发生。 如果计数器的值非零， 那么 await会一直阻塞直到计数器为零， 或者等待中的线程中断， 或者等待超时。

```java
public class TestHarness {
  public long timeTasks(int nThreads, final Runnable task) throws InterruptedException {
    final CountDownLatch startGate = new CountDownLatch(1); 
    final CountDownLatch endGate = new CountDownLatch(nThreads);
    for (int i = 0; i < nThreads; i++) { 
      Thread t = new Thread() {
        public void run() { 
          try {
            startGate.await();
            try {
              task.run();
            } finally {
              endGate.countDown();
            }
        	} catch (InterruptedException ignored) { } 
        }
      };
    	t.start(); 
    }
    
    long start = System.nanoTime(); 
    startGate.countDown(); 
    endGate.await();
    long end = System.nanoTime(); 
    return end-start;
  } 
}
```

## 2. FutureTask

FutureTask 也可以用做闭锁。FutureTask表示的计算是通过Callable 来实现的， 相当于一种可生成结果的Runnable, 并且可以处于以下3 种状态：
1. 等待运行(Waiting to run)
2. 正在运行(Running) 
3. 运行完成(Completed)

```java
public class Preloader {
  private final FutureTask<ProductInfo> future = 
    FutureTask<ProductInfo>(new Callable<ProductInfo>() { 
      public ProductInfo call() throws DataLoadException {
        return loadProductInfo(); 
      }
    });
  
  private final Thread thread = new Thread(future);   
  
  public void start() { thread.start(); }
  
  public ProductInfo get()
    				throws DataLoadException, InterruptedException {
    try {
      return future.get();
    } catch (ExecutionException e) {
      Throwable cause = e.getCause();
      if (cause instanceof DataLoadException)
        throw (DataLoadException) cause;
      else
        throw launderThrowable(cause);
    }
  }
}
```

## 3. 信号量

计数信号量(Counting Semaphore) 用来控制同时访问某个特定资源的操作数量，或者同时执行某个指定操作的数量。计数信号量还可以用来实现某种资源池， 或者对容器施加边界。

Semaphore 中管理着一组虚拟的许可(permit), 许可的初始数量可通过构造函数来指定。在执行操作时可以首先获得许可（只要还有剩余的许可）， 并在使用以后释放许可。如果没有许可， 那么acquire 将阻塞直到有许可（或者直到被中断或者操作超时）。release 方法将返回一个许可给信号量。计算信号量的一种简化形式是二值信号量， 即初始值为1 的Semaphore。二值信号量可以用做互斥体(mutex), 并具备不可重入的加锁语义： 谁拥有这个唯一的许可， 谁就拥有了互斥锁。

```java
public class BoundedHashSet<T> { 
  private final Set<T> set; 
  private final Semaphore sem;
  
  public BoundedHashSet(int bound) {
		this.set = Collections.synchronizedSet(new HashSet<T>()); 
    sem = new Semaphore(bound);
	}
  
  public boolean add(T o) throws InterruptedException { 
    sem.acquire();
  	boolean wasAdded = false; 
    try {
  		wasAdded = set.add(o);
      return wasAdded;
    }
    finally {
      if (!wasAdded)
      	sem.release();
    } 
  }
  
  public boolean remove(Object o) { 
    boolean wasRemoved = set.remove(o); 
    if (wasRemoved)
      sem.release();
		return wasRemoved;
  } 
}
```






### 未完待续。。。。。





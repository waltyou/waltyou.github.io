---
layout: post
title: Java 并发编程实战-学习日志（四）2：自定义同步器
date: 2020-01-08 17:11:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

类库中包含了许多存在状态依赖的类，例如FutureTask、Semaphore和BlockingQueue，他们的一些操作都有前提条件，例如非空，或者任务已完成等。

创建状态依赖类的最简单的方法就是在JDK提供了的状态依赖类基础上构造。例如第八章的ValueLactch，如果这些不满足，可以使用Java语言或者类库提供的底层机制来构造，包括：
- 内置的条件队列
- condition
- AQS

<!-- more -->

---

* 目录
{:toc}
---

## 状态依赖性的管理 State Dependence

下面介绍通过轮询和休眠的方式（勉强）地解决阻塞线程运行的问题。

下面是一个标准的模板，

```java
void blockingAction() throws InterruptedException {
   acquire lock on object state
   while (precondition does not hold) {
      release lock
      wait until precondition might hold
      optionally fail if interrupted or timeout expires
      reacquire lock
   }
   perform action
}
```

下面介绍阻塞有界队列的集中实现方式。依赖的前提条件是：

- 不能从空缓存中获取元素
- 不能将元素放入已满的缓存中

不满足条件时候，依赖状态的操作可以

- 抛出异常
- 返回一个错误状态（码）
- 阻塞直到进入正确的状态

下面是基类，线程安全，但是非阻塞。

```java
@ThreadSafe
public abstract class BaseBoundedBuffer <V> {
    @GuardedBy("this") private final V[] buf;
    @GuardedBy("this") private int tail;
    @GuardedBy("this") private int head;
    @GuardedBy("this") private int count;

    protected BaseBoundedBuffer(int capacity) {
        this.buf = (V[]) new Object[capacity];
    }

    protected synchronized final void doPut(V v) {
        buf[tail] = v;
        if (++tail == buf.length)
            tail = 0;
        ++count;
    }

    protected synchronized final V doTake() {
        V v = buf[head];
        buf[head] = null;
        if (++head == buf.length)
            head = 0;
        --count;
        return v;
    }

    public synchronized final boolean isFull() {
        return count == buf.length;
    }

    public synchronized final boolean isEmpty() {
        return count == 0;
    }
}
```

“先检查再运行”的逻辑解决方案如下，调用者必须自己处理前提条件失败的情况。当然也可以返回错误消息。

```java
@ThreadSafe
        public class GrumpyBoundedBuffer <V> extends BaseBoundedBuffer<V> {
    public GrumpyBoundedBuffer() {
        this(100);
    }

    public GrumpyBoundedBuffer(int size) {
        super(size);
    }

    public synchronized void put(V v) throws BufferFullException {
        if (isFull())
            throw new BufferFullException();
        doPut(v);
    }

    public synchronized V take() throws BufferEmptyException {
        if (isEmpty())
            throw new BufferEmptyException();
        return doTake();
    }
}

class ExampleUsage {
    private GrumpyBoundedBuffer<String> buffer;
    int SLEEP_GRANULARITY = 50;

    void useBuffer() throws InterruptedException {
        while (true) {
            try {
                String item = buffer.take();
                // use item
                break;
            } catch (BufferEmptyException e) {
                Thread.sleep(SLEEP_GRANULARITY);
            }
        }
    }
}
```

当然调用者可以不Sleep，而是直接重试，这种方法叫做**忙等待或者自旋等待（busy waiting or spin waiting. ）**，如果换成很长时间都不变，那么这将会消耗大量的CPU时间！！！所以调用者自己休眠，sleep让出CPU。但是这个时间就很尴尬了，sleep长了万一一会前提条件就满足了岂不是白等了从而响应性低，sleep短了浪费CPU时钟周期。另外可以试试yield，但是这也不靠谱。

下一步改进下，首先让客户端舒服些。

```java
@ThreadSafe
public class SleepyBoundedBuffer <V> extends BaseBoundedBuffer<V> {
    int SLEEP_GRANULARITY = 60;

    public SleepyBoundedBuffer() {
        this(100);
    }

    public SleepyBoundedBuffer(int size) {
        super(size);
    }

    public void put(V v) throws InterruptedException {
        while (true) {
            synchronized (this) {
                if (!isFull()) {
                    doPut(v);
                    return;
                }
            }
            Thread.sleep(SLEEP_GRANULARITY);
        }
    }

    public V take() throws InterruptedException {
        while (true) {
            synchronized (this) {
                if (!isEmpty())
                    return doTake();
            }
            Thread.sleep(SLEEP_GRANULARITY);
        }
    }
}
```

这种方式测试失败，那么释放锁，让别人做，自己休眠下，然后再检测，不断的重复这个过程，当然可以解决，但是还是需要做权衡，CPU使用率与响应性之间的抉择。

那么我们想如果这种轮询和休眠的dummy方式不用，而是存在某种挂起线程的方案，并且这种方法能够确保当某个条件成真时候立刻唤醒线程，那么将极大的简化实现工作，这就是条件队列的实现。

Condition Queues的名字来源：it gives a group of threads called the **wait set** a way to wait for a specific condition to become true. Unlike typical queues in which the elements are data items, the elements of a condition queue are the threads waiting for the condition.

每个Java对象都可以是一个锁，每个对象同样可以作为一个条件队列，并且Object的wait、notify和notifyAll就是内部条件队列的API。对象的内置锁（intrinsic lock ）和内置条件队列是关联的，**要调用X中的条件队列的任何一个方法，都必须持有对象X上的锁。**

Object.wait自动释放锁，并且请求操作系统挂起当前线程，从而其他线程可以获得这个锁并修改对象状态。当被挂起的线程唤醒时。它将在返回之前重新获取锁。

```java
@ThreadSafe
public class BoundedBuffer <V> extends BaseBoundedBuffer<V> {
    // CONDITION PREDICATE: not-full (!isFull())
    // CONDITION PREDICATE: not-empty (!isEmpty())
    public BoundedBuffer() {
        this(100);
    }

    public BoundedBuffer(int size) {
        super(size);
    }

    // BLOCKS-UNTIL: not-full
    public synchronized void put(V v) throws InterruptedException {
        while (isFull())
            wait();
        doPut(v);
        notifyAll();
    }

    // BLOCKS-UNTIL: not-empty
    public synchronized V take() throws InterruptedException {
        while (isEmpty())
            wait();
        V v = doTake();
        notifyAll();
        return v;
    }

    // BLOCKS-UNTIL: not-full
    // Alternate form of put() using conditional notification
    public synchronized void alternatePut(V v) throws InterruptedException {
        while (isFull())
            wait();
        boolean wasEmpty = isEmpty();
        doPut(v);
        if (wasEmpty)
            notifyAll();
    }
}
```

注意，如果某个功能无法通过“轮询和休眠"来实现，那么条件队列也无法实现。



## 使用条件队列

###  1. 条件谓词 The Condition Predicate

The Condition Predicate 是使某个操作成为状态依赖操作的前提条件。take方法的条件谓词是”缓存不为空“，take方法在执行之前必须首先测试条件谓词。同样，put方法的条件谓词是”缓存不满“。

在条件等待中存在一种重要的三元关系，包括

- 加锁
- wait方法
- 条件谓词

条件谓词中包含多个状态变量，而状态变量由一个锁来保护，因此在测试条件谓词之前必须先持有这个锁。锁对象和条件队列对象必须是同一个对象。wait释放锁，线程挂起阻塞，等待知道超时，然后被另外一个线程中断或者被一个通知唤醒。唤醒后，wait在返回前还需要重新获取锁，当线程从wait方法中唤醒，它在重新请求锁时不具有任何特殊的优先级，和其他人一起竞争。

### 2. 过早唤醒

其他线程中间插足了，获取了锁，并且修改了遍历，这时候线程获取锁需要重新检查条件谓词。

当然有的时候，比如一个你根本不知道为什么别人调用了notify或者notifyAll，也许条件谓词压根就没满足，但是线程还是获取了锁，然后test条件谓词，释放锁，其他线程都来了这么一趟，发生这就是“谎报军情”啊。

基于以上这两种情况，都必须重新测试条件谓词。

```java
void stateDependentMethod() throws InterruptedException {
 // condition predicate must be guarded by lock
 synchronized(lock) {  
     while (!conditionPredicate())  //一定在循环里面做条件谓词
         lock.wait();  //确保和synchronized的是一个对象
     // object is now in desired state  //不要释放锁
 }
} 
```

### 3. 丢失的信号

保证notify一定在wait之后。

### 4. 通知

调用 notify 和 notifyAll 也得持有与条件队列对象相关联的锁。调用notify，JVM Thread Scheduler在这个条件队列上等待的多个线程中选择一个唤醒，而notifyAll则会唤醒所有线程。因此一旦notify了那么就需要尽快的释放锁，否则别人都竞争等着拿锁，都会进行blocked的状态，而不是线程挂起waiting状态，竞争都了不是好事，但是这是你考了性能因素和安全性因素的一个矛盾，具体问题要具体分析。

下面的方法可以进来减少竞争，但是确然程序正确的实现有些难写，所以这个折中还得自己考虑：

```java
public synchronized void alternatePut(V v) throws InterruptedException {
        while (isFull())
            wait();
        boolean wasEmpty = isEmpty();
        doPut(v);
        if (wasEmpty)
            notifyAll();
    }
```

使用notify容易丢失信号，所以大多数情况下用notifyAll，比如take notify，却通知了另外一个take，没有通知put，那么这就是信号丢失，是一种“被劫持的”信号。

因此只有满足下面两个条件，才能用notify，而不是notifyAll：

- 所有等待线程的类型都相同
- 单进单出


### 5. 示例：阀门类A Gate Class

和第5章的那个TestHarness中使用CountDownLatch类似，完全可以使用wait/notifyAll做阀门。

```java
@ThreadSafe
public class ThreadGate {
    // CONDITION-PREDICATE: opened-since(n) (isOpen || generation>n)
    @GuardedBy("this") private boolean isOpen;
    @GuardedBy("this") private int generation;

    public synchronized void close() {
        isOpen = false;
    }

    public synchronized void open() {
        ++generation;
        isOpen = true;
        notifyAll();
    }

    // BLOCKS-UNTIL: opened-since(generation on entry)
    public synchronized void await() throws InterruptedException {
        int arrivalGeneration = generation;
        while (!isOpen && arrivalGeneration == generation)
            wait();
    }
}
```



## Explicit Condition Objects

Lock是一个内置锁的替代，而Condition也是一种广义的**内置条件队列**。

Condition的API如下：

```java
public interface Condition {
  void await() throws InterruptedException;
  boolean await(long time, TimeUnit unit)throws InterruptedException;
  long awaitNanos(long nanosTimeout) throws InterruptedException;
  void awaitUninterruptibly();
  boolean awaitUntil(Date deadline) throws InterruptedException;
  void signal();
  void signalAll();
}
```

内置条件队列存在一些缺陷，每个内置锁都只能有一个相关联的条件队列，记住是**一个**。所以在BoundedBuffer这种类中，**多个线程可能在同一个条件队列上等待不同的条件谓词**，所以notifyAll经常通知不是同一个类型的需求。如果想编写一个带有多个条件谓词的并发对象，或者想获得除了条件队列可见性之外的更多的控制权，可以使用Lock和Condition，而不是内置锁和条件队列，这更加灵活。

一个Condition和一个lock关联，想象一个条件队列和内置锁关联一样。在Lock上调用newCondition就可以新建无数个条件谓词，这些condition是可中断的、可有时间限制的，公平的或者非公平的队列操作。

下面的例子就是改造后的BoundedBuffer:

```java
@ThreadSafe
public class ConditionBoundedBuffer <T> {
    protected final Lock lock = new ReentrantLock();
    // CONDITION PREDICATE: notFull (count < items.length)
    private final Condition notFull = lock.newCondition();
    // CONDITION PREDICATE: notEmpty (count > 0)
    private final Condition notEmpty = lock.newCondition();
    private static final int BUFFER_SIZE = 100;
    @GuardedBy("lock") private final T[] items = (T[]) new Object[BUFFER_SIZE];
    @GuardedBy("lock") private int tail, head, count;

    // BLOCKS-UNTIL: notFull
    public void put(T x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)
                notFull.await();
            items[tail] = x;
            if (++tail == items.length)
                tail = 0;
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    // BLOCKS-UNTIL: notEmpty
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await();
            T x = items[head];
            items[head] = null;
            if (++head == items.length)
                head = 0;
            --count;
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```

注意这里使用了signal而不是signalll，能极大的减少每次缓存操作中发生的上下文切换和锁请求次数。

使用condition和内置锁和条件队列一样，必须保卫在lock里面。

## Synchronizer剖析

看似ReentrantLock和Semaphore功能很类似，每次只允许一定的数量线程通过，到达阀门时

- 可以通过 lock或者acquire
- 等待，阻塞住了
- 取消tryLock，tryAcquire
- 可中断的，限时的
- 公平等待和非公平等待

下面的程序是使用Lock做一个Mutex, 也就是持有一个许可的Semaphore。

```java
@ThreadSafe
public class SemaphoreOnLock {
    private final Lock lock = new ReentrantLock();
    // CONDITION PREDICATE: permitsAvailable (permits > 0)
    private final Condition permitsAvailable = lock.newCondition();
    @GuardedBy("lock") private int permits;

    SemaphoreOnLock(int initialPermits) {
        lock.lock();
        try {
            permits = initialPermits;
        } finally {
            lock.unlock();
        }
    }

    // BLOCKS-UNTIL: permitsAvailable
    public void acquire() throws InterruptedException {
        lock.lock();
        try {
            while (permits <= 0)
                permitsAvailable.await();
            --permits;
        } finally {
            lock.unlock();
        }
    }

    public void release() {
        lock.lock();
        try {
            ++permits;
            permitsAvailable.signal();
        } finally {
            lock.unlock();
        }
    }
}
```

实际上很多J.U.C下面的类都是基于AbstractQueuedSynchronizer (AQS)构建的，例如CountDownLatch, ReentrantReadWriteLock, SynchronousQueue,and FutureTask（java7之后不是了）。AQS解决了实现同步器时设计的大量细节问题，例如等待线程采用FIFO队列操作顺序。AQS不仅能极大极少实现同步器的工作量，并且也不必处理竞争问题，基于AQS构建只可能在一个时刻发生阻塞，从而降低上下文切换的开销，提高吞吐量。在设计AQS时，充分考虑了可伸缩性，可谓大师Doug Lea的经典作品啊！





## 未完待续。。。。。
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

## 未完待续。。。。。
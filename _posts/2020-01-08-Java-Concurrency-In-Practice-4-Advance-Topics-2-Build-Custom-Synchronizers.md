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



## 未完待续。。。。。
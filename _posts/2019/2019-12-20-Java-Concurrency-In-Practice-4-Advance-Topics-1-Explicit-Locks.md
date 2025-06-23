---
layout: post
title: Java 并发编程实战-学习日志（四）1：显式锁
date: 2019-12-20 17:11:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

在Java5.0之前，在协调对共享对象的访问时可以使用的机制只有synchronized和volatile。Java5.0增加了一种新的机制：ReentrantLock。与之前提到过的机制相反，ReentrantLock并不是一种替代内置加锁的方法，而是当内置加锁机制不适应时，作为一种可选的高级功能。

<!-- more -->

---

* 目录
{:toc}
---

## Lock与ReentrantLock

与内置加锁机制不同的是，Lock提供了一种无条件的、可轮询的、定时的以及可中断的锁获取操作，所有加锁和解锁的方法都是显式的。在Lock的实现中必须提供与内部锁相同的内存可见性语义，但在加锁语义、调度算法、顺序保证以及性能特性等方面可以有所不同。

```
public interface Lock {
  void lock();
  void lockInterruptibly() throws InterruptedException;
  boolean tryLock();
  boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
  void unlock();
  Condition newCondition();
}
```

ReentrantLock实现了Lock接口，并提供了与synchronized相同的排斥性和内存可见性。在获取ReentrantLock时，有着与进入同步代码块相同的内存语义，在释放ReentrantLock时，同样有着与退出同步代码块相同的内存语义。此外，与synchronized一样，ReentrantLock还提供了可重入的加锁语义。ReentrantLock支持在Lock接口中定义的多有获取锁模式，并且与synchronized相比，它还为处理锁的不可用性问题提供了更高的灵活性。

为什么要创建一种与内置锁如此相似的新加锁机制？在大多数情况下，内置锁都能很好地工作，但在功能上存在一些局限性，例如，无法中断一个正在等待获取锁的线程，或者无法在请求获取一个锁时无限地等待下去。内置锁必须在获取该锁的代码块中释放，这就简化了编码工作，并且与异常处理操作实现了很好的交互，但却无法实现非阻塞结构的加锁规则。这些都是使用synchronized的原因，但在某些情况下，一种更灵活的加锁机制通常能提供更好的活跃性或性能。

Lock锁的使用比使用内置锁复杂一些：必须在finally块中释放锁。否则，如果在被保护的代码中抛出了异常，那么这个锁永远都无法释放。

### 1. 轮询锁与定时锁

可定时的与可轮询的锁获取模式是由tryLock方法实现的，与无条件的锁获取模式相比，它具有更完善的错误恢复机制。在内置锁中，死锁是一个严重的问题，恢复程序的唯一方法是重新启动程序，而防止死锁的唯一方法就是在构造过程时避免不一致的锁顺序。可定时的与可轮询的锁提供了另一种选择：避免死锁的发生。

如果不能获得所需要的锁，那么可以使用可定时的或可轮询的锁获取方式，从而使你重新获得控制权，它会释放已经获得的锁，然后重新尝试获取所有锁（失败最好记下日志）。

#### 轮循锁

```java
while (true) {
  if (fromAcct.lock.tryLock()) {
      try {
        if (toAcct.lock.tryLock()) {
          try {
            if (fromAcct.getBalance().compareTo(amount) < 0)
              throw new InsufficientFundsException();
            else {
              fromAcct.debit(amount);
              toAcct.credit(amount);
              return true;
            }
          } finally {
            toAcct.lock.unlock();
          }
        }
      } finally {
        fromAcct.lock.unlock();
      }
    }
    if (System.nanoTime() < stopTime)
      return false;
    NANOSECONDS.sleep(fixedDelay + rnd.nextLong() % randMod);
  }
}
```

#### 定时锁

```java
public boolean trySendOnSharedLine(String message,
                                       long timeout, TimeUnit unit)
            throws InterruptedException {
  long nanosToLock = unit.toNanos(timeout) - estimatedNanosToSend(message);
  if (!lock.tryLock(nanosToLock, NANOSECONDS))
    return false;
  try {
    return sendOnSharedLine(message);
  } finally {
    lock.unlock();
  }
}
```



### 2. 可中断的锁获取操作

lockInterruptible方法能够在获得锁的同时保持对中断的响应，并且由于它包含在Lock中，因此无需创建其他类型的不可中断阻塞机制。

### 3. 非块结构的加锁

在内置锁中，锁的获取和释放等操作都是基于代码块的——释放锁的操作总是与获取锁的操作处于同一个代码块，而不考虑控制权如何退出该代码块。自动的锁释放操作简化了对程序的分析，避免了可能的编码错误，但有时候需要更灵活的加锁规则。

## 性能考虑因素

ReentrantLock能比内置锁提供更好的竞争性能。对于同步原语来说，竞争性能是可伸缩性的关键要素：如果有越多的资源被耗费在锁的管理和调度上，那么应用程序得到的资源就越少。锁的实现方式越好，将需要越少的系统调用和上下文切换，并且在共享内存总线上的内存同步通信量也越少，而一些耗时的操作将占用应用程序的计算资源。

Java6使用了改进的算法来管理内置锁，与在ReentrantLock中使用的算法类似，该算法有效地提高了可伸缩性。

## 公平性

**公平锁——Lock fairLock = new ReentrantLock(true);** 

- 在公平的锁上，线程将按照它们发出请求的顺序来获得锁
- 在非公平的锁上，则允许”插队“：当一个线程请求非公平的锁时，如果在发出请求的同时该锁的状态变为可用，那么这个线程将跳过队列中所有的等待线程并获得这个锁
- **公平性将由于在挂起线程和恢复线程时存在的开销而极大地降低性能（非公平性的锁允许线程在其他线程的恢复阶段进入加锁代码块）**
- **当持有锁的时间相对较长，或者请求锁的平局时间间隔较长，那么应该使用公平锁**
- **内置锁默认为非公平锁**

## 在Synchronized和ReentrantLock之间作出选择

在一些内置锁无法满足需求的情况下，ReentrantLock可以作为一种高级工具。当需要一些高级功能时才应该使用ReentrantLock，这些功能包括：可定时的、可轮询的与可中断的锁获取操作，公平队列，以及非块结构的锁。否则，还是应该优先使用synchronized

Synchronized是JVM的内置属性，能执行一些优化，并且基于块结构与特定栈管理，便于检测识别发生死锁。

Java6提供了一个管理和调试接口，锁可以通过该接口进行注册，从而与ReentrantLocks相关的加锁信息就能出现在转储中，并通过其他的管理接口和调试接口来访问。



## 读—写ReadWriteLock锁

一个资源可以被多个读操作访问，或者被一个写操作访问，但两者不能同时进行。

对于在多处理器系统上被频繁读取的数据结构，读 - 写锁能够提高性能。而在其他情况下，读 - 写锁的性能比独占锁的性能要略差一些，这是因为它们的复杂性更高。

```java
public interface ReadWriteLock {
  Lock readLock();
  Lock writeLock();
}
```

读写锁的可选实现：

- **释放优先**。写入锁释放后，应该优先选择读线程，写线程，还是最先发出请求的线程
- **读线程插队。**锁由读线程持有，写线程再等待，再来一个读线程，是继续让读线程访问，还是让写线程访问
- **重入性。**读取锁和写入锁是否可重入
- **降级。**将写入锁降级为读取锁
- **升级。**将读取锁升级为写入锁

```java
public class ReadWriteMap<K,V> {
  private final Map<K,V> map;
  private final ReadWriteLock lock = new ReentrantReadWriteLock(); 
  private final Lock r = lock.readLock();
  private final Lock w = lock.writeLock();
  public ReadWriteMap(Map<K,V> map) { 
    this.map = map;
  }
  public V put(K key, V value) { w.lock();
    try {
    	return map.put(key, value);
    } finally { 
      w.unlock();
    } 
	}
  // Do the same for remove(), putAll(), clear()
  public V get(Object key) { 
    r.lock();
  	try {
  		return map.get(key);
  	} finally { 
      r.unlock();
  	} 
	}
  // Do the same for other read-only Map methods
}
```

在非公平的锁中，线程获得访问许可的顺序是不确定的。写线程降级为读线程是可以的，当从读线程升级为写线程这是不可以的（这样会导致死锁）。
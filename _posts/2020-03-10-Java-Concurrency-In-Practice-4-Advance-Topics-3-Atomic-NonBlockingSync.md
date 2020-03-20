---
layout: post
title: Java 并发编程实战-学习日志（四）2：自定义同步器
date: 2020-03-10 18:11:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

"与基于锁的方案相比，非阻塞算法在设计和实现上都要复杂的多，但他们在可伸缩性和活跃性上却拥有巨大的优势。
由于非阻塞算法可以使用多个线程在竞争相同的数据时不会发生阻塞，因此它能在粒度更细的层次上进行协调。"

<!-- more -->

---

* 目录
{:toc}
---

## 锁的劣势

现代的许多JVM都对**非竞争锁获取**和**锁释放**等操作进行了极大的优化，但如果有多个线程同时请求锁，那么JVM就需要借助操作系统的功能。如果出现了这种情况，那么一些线程将被挂起并且在稍后恢复运行。

当线程恢复执行时，必须等待其他线程执行完他们的时间以后，才能被调度执行。**在挂起和恢复线程等过程中存在很大的开销，并且通常存在这个较长时间的中断**。如果在基于锁的类包中包含有细粒度的操作，(例如同步容器类，在其大多数方法中只包含少量的并发操作),那么当在锁上存在激烈的竞争时，调度开销和工作开销的比值会非常高。

与锁相比，volatile变量是一种更轻量级的同步机制，因为在使用这些变量时不会发生上下文切换和线程调度等操作。然而，volatile变量同样存在一些局限:虽然和锁机制相似的都提供了可见性保证，但是不能用于构建原子的复合操作。因此，当一个变量依赖其他变量时，或者当变量的新值依赖旧值是，就不能使用volatile变量。这些都限制了volatile变量的使用，因此volatile不能用来实现一些常用的操作，比如计数器或者互斥量。

锁定还存在其他的一些缺点。但一个线程正在等待锁时，他不能做任何其他事情。如果一个线程在持有锁的情况下被延迟执行(例如发生了缺页错误，调度延迟，或者其他类似情况)，那么所有需要这个锁的线程都无法执行下去。如果被阻塞线程的优先级很高，而持有锁的线程有限级较低，那么这个将是一个很严重的问题——也被称为优先级反转(Priority Inversion)。即使高有限级的线程可以抢先执行，但是仍然需要等待锁被释放，从而导致它的有限级会降至低优先级线程的级别。如果持有锁的线程被永久的阻塞(例如出现了无线循环，死锁，活锁或者其他活跃性故障)，所有等待这个锁的线程就会永远无法执行下去。

即使忽略这些风险，锁定方式对于细粒度的操作(例如递增计数器)来说任然是一种高开消的机制。在管理线程之间的竞争是应该有一种粒度更细的技术，类似于volatile变量的机制，同时还要支持原子的更新操作。

## 硬件对并发的支持

独占锁是一项悲观的技术——他假设最坏的情况(如果你不锁门，那么捣蛋鬼就会闯入并搞破坏)，并且只有在确保其他线程不会造成干扰(通过获取正确的锁)的情况下才能执行下去。

对于细粒度的操作，还有另外一种更高效的方法，也是一种乐观的方法。通过这种方法可以在不发生干扰的情况下完成更新操作。这种方式需要借助冲出检查机制来判断在更新的过程中是否存在来自其他线程的干扰，如果存在，这个操作将会失败，并且可以c重试(也可以不重试)。这种乐观的方法就好像一句谚语:”原谅被准许更容易得到”，其中”更容易”在这里相当于”“更高效”。

在针对多处理器操作而设计的处理器中提供了一些特殊的指令，用于管理对共享数据的并发操作。在早起的处理器初中支持原子的测试并设置(Test-and-Set),获取并递增(Fetch-and-Increment)以及交换(Swap)等指令，这些指令足以实现各种互斥量，而这些互斥量又可以实现更复杂的并发对象。现在几乎所有的现代处理器中都包含了某种形式的原子读-改-写执行，例如比较并交换(Compare-and-Swap)或者关联加载/条件存储(Load-Linked/Store-Conditional),操作系统和JVM使用这些指令来实现锁和并发的数据结构，但在Java 5.0之前，在Java类中还不能直接使用这些指令。

### 1. 比较并交换(CAS)

CAS包含三个操作数——需要读写的内存位置V，进行比较的值A和待写入的新值B。当且仅当V的值等于A时，CAS才会通过原子方式用新值 B 来更新V的值，否则不会执行任何操作。无论位置V的值是否等于A，都将返回V原有的值。

```java
@ThreadSafe
public class SimulatedCAS {
    @GuardedBy("this") private int value;

    public synchronized int get() {
        return value;
    }

    public synchronized int compareAndSwap(int expectedValue, int newValue) {
        int oldValue = value;
        if (oldValue == expectedValue)
            value = newValue;
        return oldValue;
    }

    public synchronized boolean compareAndSet(int expectedValue, int newValue) {
        return (expectedValue
                == compareAndSwap(expectedValue, newValue));
    }
}
```

## 2. 非阻塞的计数器

下面代码中的 CasCounter 使用CAS实现了一个线程安全的技术器。递增操作采用了标准形式——读取旧的值，根据它计算出新值(加1)，并使用CAS来设置这个新值。如果CAS失败，那么改操作将立即重试。通常，反复的重试是一种合理的策略，但是存在一些竞争很激烈的情况下，更好的方式是在重试之前首先等待一段时间或者回退，从而避免造成活锁问题。

```java
@ThreadSafe public class CasCounter { 
  private SimulatedCAS value; 
  
  public int getValue() { 
    return value.get(); 
  } 
  
  public int increment() { 
    int v; 
    do { 
      v = value.get(); 
    } while (v != value.compareAndSwap(v, v + 1)); 
    return v + 1; 
  } 
}
```

初看起来，基于CAS的技术器似乎比基于锁的计数器在性能上更差一些，因为他需要执行更多的操作和更复杂的控制流，并且还依赖看似复杂的CAS操作。但实际上，当实际上竞争程度不高时，基于CAS的计数器在性能上远远超过了基于锁的计数器，而且在没有竞争是甚至更高。

虽然Java语言的锁定语法比较简洁，但JVM和操作在管理锁时需要完成的工作却并不简单。在实现锁定时需要遍历JVM中一条非常复杂的代码路径，并可能导致操作系统级的锁定、线程挂起以及上下问切换等操作。在最好情况下，在锁定时至少需要一次CAS，因此虽然在使用锁时没有用到CAS，但实际上也发节约任何执行开销。另一方面，在程序内部执行CAS是不需要执行JVM代码、系统调用或者线程调度操作。在应用级看起来越长的代码路径，如国家上JVM和操作系统中的代码调用，那么事实上CAS代码却比较少。

CAS的主要**缺点**是，它将使调动者处理竞争问题(通过重试、回退、放弃)，而在锁中能自动处理竞争问题(线程在获得锁之前将一致阻塞)。

## 3. JVM对CAS操作的支持

那么，Java代码如何确保处理器执行CAS操作?在Java 5.0 之前，如果不编写明确的代码，那么就无法执行CAS。在Java 5.0 中引入了底层的支持，在int、long和对象的应用等类型上都公开了CAS操作，并且JVM把他们编译为底层硬件提供的最有效方法。在支持CAS的平台上，运行时把他们便以为相应的(多条)机器指令。

在最坏的情况下，如果不支持CAS指令，那么JVM将使用自旋锁，在原子变量类(例如java.util.concurrent.atomic中的AtomicXXX)中使用了这些底层的JVMN支持为数字类型和应用类型提供一种高效的CAS操作，二在java.util.concurrent中的大多数类在实现时则直接或者简介的使用了这些原子变量类。



## 原子变量类

原子变量比锁的粒度更细，量级更轻，并且对于在多处理器系统上实现高性能的并发代码来说是非常关键的。

原子变量类相当于一种泛化得volatile变量，能够支持原子的和有条件的读-改-写操作。AtomicInteger表示一个int类型的值，并提供了get和set方法，这些Volatile类型的int变量在读取和写入上有着相同的内存语义。它还提供了一个原子的compareAndSet方法（如果该方法成功执行，那么将实现与读取/写入一个volatile变量相同的内存效果），以及原子的增加、递增和递减等方法。

共有12个原子变量类，可分为4组：标量类（Scalar）、更新器类、数组类、复合变量类。最常用的原子变量类就是标量类：AtomicInteger、AtomicLong、AtomicBoolean、AtomicReference。所有这些类都支持CAS，此外，AtomicInteger、AtomicLong还支持算数运算。（要想模拟其它基本类型的原子变量，可以将short或byte等类型与int类型进行转换，以及使用floatToIntBits或doubleToLongBits来转换浮点数）

原子数组类（只支持Integer、Long和Reference）中的元素可以实现原子更新。原子数组类为数组的元素提供了volatile类型的访问语义，这是普通数组所不具备的特性--volatile类型的数组仅在数组引用上具有volatile语义，而在其元素上则没有。

尽管原子的标量类扩展了Number类，但并没有扩展一些基本类型的包装类，例如Integer或Long。事实上，它们也不能进行扩展：基本类型的包装类是不可修改的，而原子变量类是可以修改的。在原子变量类中同样没有重新定义hashCode和equals方法，每个实例都是不同的。与其它可变对象相同，它们也不宜用做基于散列的容器中的键值。

## 1. 原子变量是一种“更好的volatile”

通过CAS来维持包含多个变量的不变性条件

```java
public class CasNumberRange {
    private static class IntPair{
        // 不变性条件: lower <= upper
        final int lower;
        final int upper;
        
        public IntPair(int lower, int upper) {
            this.lower = lower;
            this.upper = upper;
        }
    }
    
    private AtomicReference<IntPair> values = new AtomicReference<>();
    
    public int getLower(){
        return values.get().lower;
    }
    
    public int getUpper(){
        return values.get().upper;
    }
    
    public void setLower(int i){
        while (true){
            IntPair oldv = values.get();
            if (i > oldv.upper){
                throw new IllegalArgumentException("lower can't > upper");
            }
            IntPair newv = new IntPair(i, oldv.upper);
            if (values.compareAndSet(oldv, newv)){
                return;
            }
        }
    }
}
```

# 非阻塞算法

如果在某种算法中，一个线程的失败或挂起，不会导致其他线程也失败或挂起，它就叫做非阻塞算法。

算法的每个步骤中都存在某个线程能够执行下去，这称为**无锁算法**。

在非阻塞算法中通常不会出现死锁和优先级反转问题（但可能会出现饥饿和活锁问题，因为在算法中会反复地重试）。

## 1. 非阻塞的栈

创建非阻塞算法的关键在于，找出如何将原子修改的范围缩小到单个变量上，同时还要维护数据的一致性。

非阻塞算法的特性：某项工作的完成具有不确定性，必须重新执行。

非阻塞算法中能确保线程安全性，因为compareAndSet像锁定机制一样，既能提供原子性，又能提供可见性。



## 未完待续。。。。
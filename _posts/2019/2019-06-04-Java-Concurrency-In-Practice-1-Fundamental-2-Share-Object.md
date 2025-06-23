---
layout: post
title: Java 并发编程实战-学习日志（一）2：对象的共享
date: 2019-06-04 15:03:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---



<!-- more -->

* 目录
{:toc}
---


# 可见性

可见性，又叫内存可见性（Memory Visibility）。

在单线程下，向某个变量写入值，然后在没有其他写入的情况下，我们总能可以得到相同的值。但是当写和读在不同的线程下进行时，情况却并非如此。

看如下代码：

```java
public class NoVisibility {
    private static boolean ready;
    private static int number;
    private static class ReaderThread extends Thread {
        public void run() {
            while (!ready)
            	Thread.yield();
            System.out.println(number);
        }
    }
    public static void main(String[] args) {
        new ReaderThread().start();
        number = 42;
        ready = true;
    }
}
```

ReaderThread 这个线程可能会一直运行下去，也可能会打印 0 之后退出。这显然不是我们想要的，但产生这个现象的原因是什么呢？这是因为**重排序**。

> 在没有同步的情况下，编译器、处理器以及运行时等都可能对操作的执行顺序进行一些意想不到的调整。在缺乏足够同步的多线程程序中，要想对内存操作的执行顺序进行判断，几乎无法得出正确的结论。



## 1. 失效数据

在没有同步的情况下，线程去读取变量时，可能会得到一个已经失效的值。更糟糕的是，失效值可能不会同时出现：一个线程可能获得某个变量的最新值，而获得另一个变量的失效值。

## 2. 非原子的64位操作

当线程在没有同步的情况下读取变量时，可能会得到一个失效值，但至少这个值是由之前某个线程设置的值，而不是一个随机值。这种安全性保证也被称为**最低安全性**（out-of-thin-air safety）。

最低安全性适用于绝大多数变量，当时存在一个例外：非volatile类型的64位数值变量。Java内存模型要求，变量的读取操作和写入操作都必须是原子操作，但对于非volatile类型的long和double变量，JVM允许将64位的读操作或写操作分解为两个32位的操作。

所以如果对64位的读、写操作在两个线程中，就有可能发生一个写了高32位，一个读了低32位。所以，要用 volatile 关键字来修饰它们，或者加锁。

## 3. 加锁与可见性

内置锁可以用于确保某个线程以一种可预测的方式来查看另一个线程的执行结果。

下图展示了锁是如何保证可见性的：

[![](/images/posts/Visibility_Guarantees_for_Synchronization.png)](/images/posts/Visibility_Guarantees_for_Synchronization.png)

> 加锁的含义不仅仅局限于互斥行为，还包括内存可见性。为了确保所有线程都能看到共享变量的最新值，所有执行读操作或者写操作的线程都必须在同一个锁上同步。

## 4. Volatile变量

Java 提供了一种稍弱的同步机制，即 volatile 变量，用来确保将变量的更新操作通知到其他线程。当把变量声明为volatile类型后，编译器和运行时都会注意到这个变量是**共享**的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile变量不会被缓存在寄存器或者对其他处理器不可见的地方，因此读取volatile类型的变量时总会返回最新写入的值。

从内存可见性的角度看，写入volatile变量相当于退出同步代码块，而读取volatile变量相当于进入同步代码块。

> 仅当 volatile 变量能简化代码的实现时以及对同步策略的验证时，才应该使用它们。

volatile 变量正确的使用方式包括：

- 确保它们自身状态的可见性
- 确保它们所引用对象的状态的可见性
- 标识一些重要的程序生命周期事件的发生

看一个数绵羊睡觉的代码。

```java
volatile boolean asleep；
...
while(!asleep){
	contSomeSheep();
}
```

### 局限性

> 加锁机制既可以保证可见性也可以保证原子性，而volatile变量只能保证可见性。加锁等价于在锁释放的时候，自动将数据进行同步，确保了可见性。

当且仅当满足以下所有条件时，才应该使用 volatile 变量：

- 对变量的写入操作不依赖当前变量的当前值，或者你能确保只有单个线程更新变量的值
- 该变量不会与其他状态变量一起纳入不变性条件中
- 在访问变量时不需要加锁

# 发布与逸出

## 1. 发布

“发布 publish” 一个对象的意思是指，使对象能够在当前作用域之外的代码中使用。

```java
public static Set<Secret> knownSecrets;
public void initialize() {
	knownSecrets = new HashSet<Secret>();
}
```

可以是以下几种情况：

1. 将对象的引用保存到一个公有的静态变量中，以便任何类和线程都能看到该对象；
2. 发布某个对象的时候，在该对象的非私有域中引用的所有对象都会被发布；
3. 发布一个内部的类实例，内部类实例关联一个外部类引用。

## 2. 逸出

“逸出”是指某个不应该发布的对象被公布。

```java
class UnsafeStates {
    private String[] states = new String[] {
    	"AK", "AL" ...
    };
    public String[] getStates() { return states; }
}
```

某个对象逸出后，你必须假设有某个类或线程可能会误用该对象，所以要封装。

不要在构造过程中使this引用逸出。常见错误：在构造函数中启动一个线程。

### 例子

错误使用方式：隐式地使 this 引用逸出

```java
public class ThisEscape {
    public ThisEscape(EventSource source) {
        source.registerListener(
            new EventListener() {
                public void onEvent(Event e) {
                	doSomething(e);
            }
        });
    }
}
```

使用工厂方法来防止 this 引用在构造函数中逸出：

```java
public class SafeListener {
    private final EventListener listener;
    
    private SafeListener() {
        listener = new EventListener() {
            public void onEvent(Event e) {
                doSomething(e);
            }
        };
    }
    
    public static SafeListener newInstance(EventSource source) {
        SafeListener safe = new SafeListener();
        source.registerListener(safe.listener);
        return safe;
    }
}
```

# 线程封闭

数据只在单个线程中使用，不去共享数据，这就叫做线程封闭。

## 1. Ad-hoc 线程封闭

Ad-hoc线程封闭是指：维护线程封闭性的职责完全由程序实现来承担。

它是非常脆弱的，因为没有一种语言特性，能将对象封闭到目标线程上。事实上，对线程封闭对象的引用都保存在公有变量中。

## 2. 栈封闭

在栈封闭中，只能通过局部变量才能访问对象。

比如如下代码中的 `numPairs` 变量就是一个栈封闭的对象，它既是基础类型，又是局部变量。

```java
public int loadTheArk(Collection<Animal> candidates) {
    SortedSet<Animal> animals;
    int numPairs = 0;
    Animal candidate = null;
    // animals confined to method, don't let them escape!
    animals = new TreeSet<Animal>(new SpeciesGenderComparator());
    animals.addAll(candidates);
    for (Animal a : animals) {
        if (candidate == null || !candidate.isPotentialMate(a))
        	candidate = a;
        else {
            ark.load(new AnimalPair(candidate, a));
            ++numPairs;
            candidate = null;
        }
    }
    return numPairs;
}
```

## 3. ThreadLocal类

维护线程封闭更规范方法是使用 ThreadLocal 这个类，它能使线程中的某个值与保存值的对象关联起来。

它也提供 `get()` 和 `set()` 方法，这些方法为每个使用该变量的线程都存有一份独立的副本，因此`get`总是返回由当前执行线程在调用`set`时设置的最新值。

ThreadLocal对象通常用于防止对可变的单实例变量（Singleton）或全局变量进行共享。

如 JDBC 的Connection对象，为了防止共享，所以通常将JDBC的连接保存到ThreadLocal对象中，每个线程都会拥有属于自己的连接。

```java
private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<Connection>() {
    public Connection initialValue() {
    	return DriverManager.getConnection(DB_URL);
    }
};

public static Connection getConnection() {
	return connectionHolder.get();
}
```

ThreadLocal 也不能太过滥用，因为它会降低代码的可重用性，并在类之间引入隐含的耦合性。

# 不变性

> 不变的对象一定是线程安全的。

当满足以下这些条件的时候，对象才是不可变的：

- 对象创建以后其状态就不能被修改；
- 对象的所有域都是 final 类型；
- 对象是正确创建的（在对象的创建期间，this引用没有逸出）。



## 1. Final 域

final 类型的域是不能修改的，这样可以确保初始化过程中的安全性，从而可以不受限制地访问不可变对象，并在共享这些对象时无需同步。

> 正如“除非需要更高的可见性，否则所有的域都应该声明为私有域”，也应遵守 “除非需要某个域是需要变化的，否则应将其声明为 final 域”。

## 2. 示例：使用volatile 类型来发布不可变对象

对数值及其因数分解结果进行缓存的不可变容器类：

```java
@Immutable
class OneValueCache {
    private final BigInteger lastNumber;
    private final BigInteger[] lastFactors;
    public OneValueCache(BigInteger i,BigInteger[] factors) {
        lastNumber = i;
        lastFactors = Arrays.copyOf(factors, factors.length);
    }
    public BigInteger[] getFactors(BigInteger i) {
        if (lastNumber == null || !lastNumber.equals(i))
        	return null;
        else
        	return Arrays.copyOf(lastFactors, lastFactors.length);
    }
}
```

使用指向不可变容器对象的 volatile 类型引用以缓存最新的结果：

```java
@ThreadSafe
public class VolatileCachedFactorizer implements Servlet {
    
    private volatile OneValueCache cache = new OneValueCache(null, null);
    
    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = cache.getFactors(i);
        if (factors == null) {
            factors = factor(i);
            cache = new OneValueCache(i, factors);
        }
        encodeIntoResponse(resp, factors);
    }
}
```

> 注意: 
>
> 经自己测试，以上的方法只能保证 `OneValueCache` 中的 `lastNumber` 与 `lastFactors`是线程安全且一直一致的，因为它们会在构造方法中一起设置好后，才产生新的 OneValueCache 对象。但是这个代码无法保证，相同内容的 `OneValueCache` 被重复创建。



# 安全发布

## 1. 不正确的发布：正确的对象被破坏

我们不能指望一个尚未完全创建的对象拥有完整性。

在没有足够同步的情况下发布对象（不要这么做）：

```java
// Unsafe publication
public Holder holder;
public void initialize() {
    holder = new Holder(42);
}
```

由于未被正确发布，因此这个类可以出现故障：

```java
public class Holder {
    private int n;
    
    public Holder(int n) { this.n = n; }
    
    public void assertSanity() {
        if (n != n)
        	throw new AssertionError("This statement is false.");
    }
}
```

故障可能有如下几种：

1. 除了发布线程外，其他线程看到的 Holder 是一个空引用或者之前的旧值
2. 某个线程第一次读取 n 时得到的是失效值，而再次读取时会得到一个更新值，所以 `assertSanity` 会抛出 `AssertionError` 。

## 2. 不可变对象与初始化安全性

任何线程都可以在不需要额外同步的情况下，安全地访问不可变对象，即使在发布这些对象时没有使用同步。

## 3. 安全发布的常用模式

要安全的发布一个对象，对象的引用以及对象的状态必须同时对其它线程可见。一个正确构造的对象可以通过以下方式来安全地发布：

- 在静态初始化函数中初始化一个对象引用。
- 将对象的引用保存到volatile类型的于或者AtomicReferance对象中。
- 将对象的引用保存到某个正确构造对象的final类型域中。
- 将对象的引用保存到一个由锁保护的域中。

## 4. 事实不可变对象

如果对象从技术上来看是可变的，但其状态在发布后不会再改变，那么把这种对象成为“事实不可变对象”。

使用这些对象，不仅可以简化开发过程，也能减少同步而提高性能。

> 在没有额外的同步的情况下，任何线程都可以安全地使用被安全发布的事实不可变对象。

## 5. 可变对象

对象的发布需求取决于它的可变性：

1. 不可变对象可以通过任意机制来发布
2. 事实不可变对象必须通过安全的发布机制来发布
3. 可变对象必须通过安全的发布机制来发布，并且必须是线程安全的或者有某个锁保护起来

## 6. 安全地共享对象

并发编程中共享对象的一些使用策略：

### 线程封闭

线程封闭的对象只能由一个线程拥有，对象被封闭在该线程中，并且只能由这个线程修改。

### 只读共享

在没有额外同步的情况下，共享的只读对象可以有多个线程并发访问，但任何线程都不能修改它。共享的只读对象包括不可变对象和事实不可变对象。

### 线程安全共享

线程安全的对象在其内部实现同步，因此多个线程可以通过对象的公有接口进行访问而不需要进一步的同步。

### 保护对象

被保护的对象只能通过持有特定的锁来访问。保护对象包括封装在其他安全对象中的对象，以及已发布的并且有某个特定的锁保护的对象。


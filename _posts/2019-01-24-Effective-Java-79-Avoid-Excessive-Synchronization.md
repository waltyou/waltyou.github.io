---
layout: post
title: 《Effective Java》学习日志（十）79:避免过度使用同步
date: 2019-01-24 14:45:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]

---



<!-- more -->

------

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

------

## 目录
{:.no_toc}

* 目录
{:toc}

------

项目78警告同步不足的危险。 这个项目涉及相反的问题。 根据具体情况，过度同步可能会导致性能降低，死锁或甚至出现不确定行为。

**为避免活动和安全故障，请勿在同步方法或块中将控制权交给客户端。** 换句话说，在同步区域内，不要调用设计为被覆盖的方法，也不要调用客户端以函数对象的形式提供的方法（第24项）。 从具有同步区域的类的角度来看，这种方法是陌生的。 该类不知道该方法的作用，也无法控制它。 根据外来方法的作用，从同步区域调用它可能会导致异常，死锁或数据损坏。

为了使这个具体，请考虑以下类，它实现了一个可观察的集合包装器。 它允许客户端在元素添加到集合时订阅通知。 这是观察者模式[Gamma95]。 为简洁起见，当从集合中删除元素时，类不提供通知，但提供它们将是一件简单的事情。 此类在第18项（第90页）中可重用的ForwardingSet顶部实现：

```java
// Broken - invokes alien method from synchronized block!
public class ObservableSet<E> extends ForwardingSet<E> {
    
    public ObservableSet(Set<E> set) { super(set); }
    
    private final List<SetObserver<E>> observers = new ArrayList<>();
    
    public void addObserver(SetObserver<E> observer) {
        synchronized(observers) {
        	observers.add(observer);
        }
    }
    public boolean removeObserver(SetObserver<E> observer) {
        synchronized(observers) {
        	return observers.remove(observer);
        }
    }
    private void notifyElementAdded(E element) {
        synchronized(observers) {
            for (SetObserver<E> observer : observers)
            	observer.added(this, element);
        }
    }
    @Override 
    public boolean add(E element) {
        boolean added = super.add(element);
        if (added)
        	notifyElementAdded(element);
        return added;
    }
    @Override 
    public boolean addAll(Collection<? extends E> c) {
        boolean result = false;
        for (E element : c)
        	result |= add(element); // Calls notifyElementAdded
        return result;
    }
}
```

观察者通过调用addObserver方法订阅通知，并通过调用removeObserver方法取消订阅。 在这两种情况下，都会将此回调接口的实例传递给该方法。

```java
@FunctionalInterface 
public interface SetObserver<E> {
	// Invoked when an element is added to the observable set
	void added(ObservableSet<E> set, E element);
}
```

该接口在结构上与 BiConsumer<ObservableSet<E>,E> 相同。 我们选择定义自定义功能接口，因为接口和方法名称使代码更具可读性，并且因为接口可以演变为包含多个回调。 也就是说，使用BiConsumer也可以做出合理的论证（议题44）。

在粗略检查中，ObservableSet似乎工作正常。 例如，以下程序打印0到99之间的数字：

```java
public static void main(String[] args) {
    ObservableSet<Integer> set = new ObservableSet<>(new HashSet<>());
    set.addObserver((s, e) -> System.out.println(e));
    for (int i = 0; i < 100; i++)
    	set.add(i);
}
```

现在让我们尝试更有趣的东西。 假设我们将一个addObserver调用替换为一个传递一个观察者的调用，该观察者打印添加到集合中的Integer值，如果值为23则自行删除：

```java
set.addObserver(new SetObserver<>() {
    public void added(ObservableSet<Integer> s, Integer e) {
        System.out.println(e);
        if (e == 23)
        	s.removeObserver(this);
        } 
	}
);
```

请注意，此调用使用匿名类实例代替上一次调用中使用的lambda。 这是因为函数对象需要将自身传递给s.removeObserver，而lambdas不能自己访问（第42项）。

您可能希望程序打印0到23的数字，之后观察者将取消订阅并且程序将以静默方式终止。 实际上，它打印这些数字然后抛出ConcurrentModificationException。 问题是notifyElementAdded在调用观察者添加的方法时正在迭代观察者列表。 添加的方法调用observable set的removeObserver方法，该方法又调用方法observers.remove。 现在我们遇到了麻烦。 我们试图在迭代它的过程中从列表中删除一个元素，这是非法的。 notifyElementAdded方法中的迭代在同步块中以防止并发修改，但它不会阻止迭代线程本身回调到可观察集并修改其观察者列表。

现在让我们尝试一些奇怪的事情：让我们编写一个尝试取消订阅的观察者，但不是直接调用removeObserver，而是使用另一个线程的服务来执行契约。 该观察者使用执行者服务（第80项）：

```java
// Observer that uses a background thread needlessly
set.addObserver(new SetObserver<>() {
    public void added(ObservableSet<Integer> s, Integer e) {
        System.out.println(e);
        if (e == 23) {
            ExecutorService exec = Executors.newSingleThreadExecutor();
            try {
                exec.submit(() -> s.removeObserver(this)).get();
            } catch (ExecutionException | InterruptedException ex) {
                throw new AssertionError(ex);
            } finally {
                exec.shutdown();
            }
        }
    }
}
);
```

顺便提一下，请注意，此程序在一个catch子句中捕获两种不同的异常类型。 Java 7中添加了这种非常称为multi-catch的工具。它可以极大地提高清晰度并减小程序的大小，这些程序在响应多种异常类型时的行为方式相同。

当我们运行这个程序时，我们没有得到例外; 我们陷入僵局。 后台线程调用s.removeObserver，它试图锁定观察者，但它无法获取锁，因为主线程已经有锁。 一直以来，主线程都在等待后台线程完成删除观察者，这解释了死锁。

这个例子是设计的，因为观察者没有理由使用后台线程来取消订阅，但问题是真实的。 从同步区域内调用外来方法已在实际系统中引起许多死锁，例如GUI工具包。

在前面的两个例子中（异常和死锁）我们很幸运。 当调用外来方法（添加）时，由同步区域（观察者）保护的资源处于一致状态。 假设您要从同步区域调用外来方法，而受同步区域保护的不变量暂时无效。 因为Java编程语言中的锁是可重入的，所以这样的调用不会死锁。 与导致异常的第一个示例一样，调用线程已经保持锁定，因此线程在尝试重新获取锁定时将成功，即使锁定保护的数据正在进行另一个概念上不相关的操作。 这种失败的后果可能是灾难性的。 从本质上讲，锁定未能完成其工作。 重入锁简化了多线程面向对象程序的构建，但它们可以将活动失败转化为安全故障。

幸运的是，通过将异步方法调用移出同步块来解决此类问题通常并不困难。 对于notifyElementAdded方法，这涉及获取观察者列表的“快照”，然后可以在没有锁定的情况下安全地遍历。 通过此更改，前面的两个示例都运行无异常或死锁：

```java
// Alien method moved outside of synchronized block - open calls
private void notifyElementAdded(E element) {
    List<SetObserver<E>> snapshot = null;
    synchronized(observers) {
    	snapshot = new ArrayList<>(observers);
    } 
    for ( SetObserver<E> observer : snapshot)
    	observer.added(this, element);
}
```

实际上，有一种更好的方法可以将异类方法调用移出synchronized块。 这些库提供了一个名为CopyOnWriteArrayList的并发集合（Item 81），它是为此目的而量身定制的。 此List实现是ArrayList的一种变体，其中所有修改操作都是通过制作整个底层数组的新副本来实现的。 因为内部数组永远不会被修改，所以迭代不需要锁定并且非常快。 对于大多数用途，CopyOnWriteArrayList的性能会很糟糕，但它非常适合观察者列表，这些列表很少被修改并经常遍历。

如果修改列表以使用CopyOnWriteArrayList，则无需更改ObservableSet的add和addAll方法。 以下是该类其余部分的外观。 请注意，没有任何明确的同步：

```java
// Thread-safe observable set with CopyOnWriteArrayList
private final List<SetObserver<E>> observers = new CopyOnWriteArrayList<>();
public void addObserver(SetObserver<E> observer) {
    observers.add(observer);
}
public boolean removeObserver(SetObserver<E> observer) {
	return observers.remove(observer);
}
private void notifyElementAdded(E element) {
    for (SetObserver<E> observer : observers)
        observer.added(this, element);
}
```

在同步区域之外调用的外来方法称为开放调用[Goetz06,10.1.4]。 除了防止失败，开放调用可以大大增加并发性。 外来方法可能会持续任意长时间。 如果从同步区域调用alien方法，则将不允许其他线程访问受保护资源。

**通常，您应该在同步区域内尽可能少地工作**。 获取锁，检查共享数据，根据需要进行转换，然后取消锁定。 如果您必须执行一些耗时的活动，请找到一种方法将其移出同步区域，而不违反第78项中的准则。

这个项目的第一部分是关于正确性。 现在让我们简要介绍一下性能。 虽然自Java早期以来同步成本已经大幅下降，但重要的是不要过度同步。 在多核世界中，过度同步的实际成本不是获得锁定所花费的CPU时间; 这是争论：并行性失去的机会以及确保每个核心都有一致的记忆观点的需要所造成的延迟。 过度同步的另一个隐藏成本是它可以限制VM优化代码执行的能力。

如果您正在编写可变类，则有两个选项：您可以省略所有同步并允许客户端在需要并发使用时从外部进行同步，或者您可以在内部进行同步，从而使类具有线程安全性（第82项）。 只有当您通过内部同步实现显着更高的并发性时，才应选择后一个选项，而不是让客户端在外部锁定整个对象。 java.util中的集合（过时的Vector和Hashtable除外）采用前一种方法，而java.util.concurrent中的集合采用后者（项目81）。

在Java的早期，许多类违反了这些准则。 例如，StringBuffer实例几乎总是由单个线程使用，但它们执行内部同步。 正是由于这个原因，StringBuffer被StringBuilder取代，而StringBuilder只是一个不同步的StringBuffer。 同样，java.util.Random中的线程安全伪随机数生成器被java.util.concurrent.ThreadLocalRandom中的非同步实现取代也是很大一部分原因。 如有疑问，请不要同步您的类，但要记录它不是线程安全的。

如果在内部同步类，则可以使用各种技术来实现高并发性，例如锁定拆分，锁定条带化和非阻塞并发控制。 

如果方法修改了静态字段，并且有可能从多个线程调用该方法，则必须在内部同步对该字段的访问（除非该类可以容忍非确定性行为）。 多线程客户端无法在此类方法上执行外部同步，因为不相关的客户端可以在不同步的情况下调用该方法。 该字段本质上是一个全局变量，即使它是私有的，因为它可以由不相关的客户端读取和修改。 第78项中方法generateSerialNumber使用的nextSerialNumber字段举例说明了这种情况。

总之，为避免死锁和数据损坏，请勿在同步区域内调用外来方法。 更一般地说，将您在同步区域内完成的工作量保持在最低水平。 在设计可变类时，请考虑是否应该进行自己的同步。 在多核时代，不要过度同步比以往任何时候都重要。 只有在有充分理由的情况下才能在内部同步您的课程，并清楚地记录您的决定（第82项）。

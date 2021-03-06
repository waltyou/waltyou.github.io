---
layout: post
title: 《Effective Java》学习日志（十）82:在文档中记录线程安全
date: 2019-02-11 10:43:00
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]

---



<!-- more -->

------

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

------




* 目录
{:toc}

------

当并发使用一个类的方法时，类的行为是其与客户签订合同的重要部分。 如果您未能记录某个类行为的这一方面，其用户将被迫做出假设。 如果这些假设是错误的，则生成的程序可能执行不充分的同步（项目78）或过度同步（项目79）。 无论哪种情况，都可能导致严重错误。

您可能会听到它说您可以通过在其文档中查找synchronized修饰符来判断方法是否是线程安全的。 这有几点是错误的。 在正常操作中，Javadoc在其输出中不包含synchronized修饰符，并且有充分的理由。 **方法声明中synchronized修饰符的存在是一个实现细节，而不是其API的一部分**。 它不能可靠地表明方法是线程安全的。

此外，声称同步修改器的存在足以记录线程安全性的说法体现了线程安全是全有或全无的属性的误解。 实际上，有几个级别的线程安全性。 **要启用安全的并发使用，类必须清楚地记录它支持的线程安全级别**。 以下列表总结了线程安全级别。 它并非详尽无遗，但涵盖了常见情况：

- **Immutable 不变的**：此类的实例显示为常量。 无需外部同步。 示例包括String，Long和BigInteger（第17项）。
- **Unconditionally thread-safe 无条件线程安全**： 此类的实例是可变的，但该类具有足够的内部同步，以便可以同时使用其实例而无需任何外部同步。 示例包括AtomicLong和ConcurrentHashMap。
- **Conditionally thread-safe 有条件线程安全**： 像无条件线程安全一样，除了某些方法需要外部同步以便安全并发使用。 示例包括Collections.synchronized包装器返回的集合，其迭代器需要外部同步。
- **Not thread-safe 不是线程安全**：这个类的实例是可变的。 要同时使用它们，客户端必须使用客户端选择的外部同步来包围每个方法调用（或调用序列）。 示例包括通用集合实现，例如ArrayList和HashMap。
- **Thread-hostile 线程敌对**： 即使每个方法调用都被外部同步包围，此类对于并发使用也是不安全的。 线程敌意通常是在没有同步的情况下修改静态数据。 没有人故意写一个线索敌对的类; 此类通常是由于未考虑并发而导致的。 当发现类或方法是线程敌对的时，通常会修复或弃用它。 如第322页所述，在没有内部同步的情况下，第78项中的generateSerialNumber方法将是线程不可靠的。

这些类别（除了线程敌对）大致对应于Java Concurrency in Practice中的线程安全注释，它们是Immutable，ThreadSafe和NotThreadSafe [Goetz06，附录A]。 上述分类中的无条件和条件线程安全类别都包含在ThreadSafe注释中。

记录有条件的线程安全类需要小心。 您必须指明哪些调用序列需要外部同步，以及必须获取哪个锁（或在极少数情况下，锁）才能执行这些序列。 通常它是实例本身的锁，但也有例外。 例如，Collections.synchronizedMap的文档说明了这一点：“当迭代任何集合视图时，用户必须手动同步返回的map”。

```java
Map<K, V> m = Collections.synchronizedMap(new HashMap<>());
Set<K> s = m.keySet(); // Needn't be in synchronized block
...
synchronized(m) { // Synchronizing on m, not s!
    for (K key : s)
    	key.f();
}
```

不遵循此建议可能会导致非确定性行为。

类的线程安全性的描述通常属于类的doc注释，但具有特殊线程安全属性的方法应在其自己的文档注释中描述这些属性。 没有必要记录枚举类型的不变性。 除非从返回类型中显而易见，否则静态工厂必须记录返回对象的线程安全性，如Collections.synchronizedMap（上文）所示。

当类承诺使用可公开访问的锁时，它允许客户端以原子方式执行一系列方法调用，但这种灵活性需要付出代价。 它与并发集合（如ConcurrentHashMap）使用的高性能内部并发控制不兼容。 此外，客户端可以通过长时间保持可公开访问的锁来发起拒绝服务攻击。 这可以是偶然或有意的。

要防止此拒绝服务攻击，您可以使用私有锁对象而不是使用synchronized方法（这意味着可公开访问的锁）：

```java
// Private lock object idiom - thwarts denial-of-service attack
private final Object lock = new Object();

public void foo() {
    synchronized(lock) {
    	...
    }
}
```

由于私有锁对象在类外是不可访问的，因此客户端不可能干扰对象的同步。 实际上，我们通过将锁定对象封装在它同步的对象中来应用第15项的建议。

请注意，锁定字段被声明为final。 这可以防止您无意中更改其内容，从而导致灾难性的非同步访问（第78项）。 我们通过最小化锁定字段的可变性来应用第17项的建议。 **锁定字段应始终声明为final**。 无论您使用普通的监视器锁（如上所示）还是使用java.util.concurrent.locks包中的锁，都是如此。

私有锁对象习惯用法只能用于无条件的线程安全类。 有条件的线程安全类不能使用这个习惯用法，因为它们必须记录在执行某些方法调用序列时客户端要获取的锁。

私有锁对象习惯用法特别适合用于继承的类（第19项）。 如果这样的类要使用其实例进行锁定，则子类可能容易且无意地干扰基类的操作，反之亦然。 通过为不同的目的使用相同的锁，子类和基类可能最终“踩到彼此的脚趾。”这不仅仅是一个理论问题; 它发生在Thread类中。

总之，每一个类应该清楚地以措辞谨慎的文字描述或线程安全的注解记录它的线程安全性。 synchronized修饰符在本文档中不起作用。 有条件的线程安全类必须记录哪些方法调用序列需要外部同步，哪些锁定在执行这些序列时获取。 如果编写无条件的线程安全类，请考虑使用私有锁对象代替同步方法。 这可以保护您免受客户端和子类的同步干扰，并使您可以更灵活地在以后的版本中采用复杂的并发控制方法。


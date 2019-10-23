---
layout: post
title: Java 并发编程实战-学习日志（三）1：避免活跃性危险
date: 2019-10-21 17:45:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

在安全性与活跃性之间通常存在着某种制衡，我们使用加锁机制来确保线程安全，但如果过度地使用加锁，则可能导致“锁顺序死锁”。同样，我们使用线程池和信号量来限制对资源的使用，但这些被限制的行为可能会导致资源死锁。

<!-- more -->

---

* 目录
{:toc}
---

# 死锁

> 每个人都拥有其他人需要的资源，同时又等待其他人已经拥有的资源，并且每个人在获得所有需要的资源之前都不会放弃已经拥有的资源。

当一个线程永远地持有一个锁，并且其他线程都尝试获得这个锁时，那么它们将永远被阻塞。这种情况就是最简单的死锁形式（称为抱死[Deadly Embrace]）,其中多个线程由于存在环路的锁依赖关系而永远等待下去。（把每个线程假想为有向图的一个节点，图中每条边表示的关系是：“线程A等待线程B所占有的资源”。如果图中形成一条环路，那么就存在一个死锁）。

## 1. 锁顺序死锁

两个线程试图以不同的顺序来获得相同的锁。如果按照相同的顺序来请求锁，那么就不会出现循环的加锁依赖性，不会产生死锁。如果每个需要锁L和锁M的线程都以相同的顺序来获取L和M，就不会发生死锁。

## 2. 动态的锁顺序死锁

考虑下方的代码，它将资金从一个账户转入到另一个账户。在开始转账之前，首先要获得这两个Account对象的锁，以却不通过原子方式来更新两个账户中的余额，同时又不能破坏一些不变性条件，例如“账户的余额不能为负数”。

```java
//容易发生死锁
public void transferMoney(Account fromAccount,
                          Account toAccount,
                          DollarAmount amount)
           throws InsufficientFundsException {
   synchronized (fromAccount) {
     synchronized (toAccount) {
        if (fromAccount.getBalance().compareTo(amount) < 0)
            throw new InsufficientFundsException();
        else {
           fromAccount.debit(amount);
           toAccount.credit(amount);
        }
    }
  }
复制代码
```

所有的线程似乎按相同的顺序来获得锁，**但事实上锁的顺序取决与传递给transferMoney的参数顺序，而这些参数顺序又取决与外部输入。** 如果两个线程同时调用transferMoney,其中一个线程从X向Y转账，而另一个线程从Y向X转账，那么就会发生死锁：

> A: transferMoney(myAccount, yourAccount, 10);
>
> B: transferMoney(yourAccount, myAccount, 20);

如果执行时序不当，那么A可能获得myAccount的锁并等待yourAccount的锁，然而B此时拥有yourAccount的锁并正在等到myAccount的锁。

在制定锁的顺序时，可以使用***System.identityHashCode***方法，该方法将返回由Object.hashCode返回的值。下方给出另一个版本的transferMoney，使用了System.identityHashCode来定义锁的顺序。 虽然加了一些新的代码，但却消除了死锁的可能性。

在极少数情况下，两个对象可能拥有相同的**散列值（HashCode）**，此时必须通过某种任意的方法来决定锁的顺序，而这可能又会重新引入死锁。为了避免这种情况，可以使用**“加时赛（Tie-Breaking）锁”**。在获得两个Account锁之前，首先获得这个“加时赛”锁，从而保证每次只有一个线程以未知的顺序得到这两个锁，从而消除了死锁发生的可能性（只要一致地使用这种机制）。

如果经常出现散列冲突（hash collisions），那么这种技术可能会称为并发性的一个瓶颈（类似与在整个程序中只有一个锁的情况，因为经常要等待获得加时赛锁），但由于System.identityHashCode中出现散列冲突的频率非常低，因此这项技术以最小的代价，换来了最大的安全性。

**如果在Account中包含一个唯一的，不可变的，并且具备可比性的键值**，例如账号，那么要指定锁的顺序就更加容易：通过键值对对象进行排序，因而不需要使用“加时赛”锁。



## 未完待续。。。。




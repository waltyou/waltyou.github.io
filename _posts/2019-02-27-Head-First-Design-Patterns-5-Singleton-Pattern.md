---
layout: post
title: 《HeadFirst设计模式》学习日志（五）:单例模式
date: 2019-02-27 10:23:00
author: admin
comments: true
categories: [Java]
tags: [Java,Design Patterns]
---



<!-- more -->

------

学习资料主要参考： 《Head First 设计模式》

------

## 目录
{:.no_toc}

* 目录
{:toc}
------

单例模式是用来创建独一无二的、只有一个实例的类。

常用来管理共享的资源，例如数据库连接或者线程池。

有许多时候，我们只需要一个对象，比如线程池、缓存、打印机等，如果有多个对象，可能会导致许多问题发生，比如程序异常、资源使用过量。

## 经典的单例模式

```java
public class Singleton {

    private static Singleton uniqueInstance;

    private Singleton() {
    }

    public static Singleton getUniqueInstance() {
        if (uniqueInstance == null) {
            uniqueInstance = new Singleton();
        }
        return uniqueInstance;
    }
}
```

注意点：

- 将类的构造器修饰符变为 private， 这样子别的地方就无法实例化这个类。
- 只能通过这个类的 *getUniqueInstance* 方法来获取对象。这个对象有可能是刚刚创建的，也可能是之前创建的。如果我们一直不调用 *getUniqueInstance* 方法，*Singleton* 对象就不会创建，这就是“延迟实例化”。

## 定义单例模式

> 确保一个类只有一个实例，并提供一个全局访问点。

## 经典单例模式的局限性

多线程下，可能无法正常工作，因为 getUniqueInstance 并不是线程安全的。

那么只要把 getUniqueInstance  变为线程安全（添加**synchronized** 关键字）的就可以解决这个局限了。

上述解决方法又引入了新的问题：

- 同步方法后，性能降低
- 其实只需要在第一次调用 getUniqueInstance  方法时进行同步即可，但是这种写法，每次调用方法都会同步，其实是一种浪费

如何改善呢？遵循一下原则：

1. 如果 getUniqueInstance 的性能对应用不是很关键，就什么都别做
2. 提前实例化好对象，而不是延迟实例化
3. 使用“双重检查加锁”，减少 getUniqueInstance  的同步

### 双重检查加锁

```java
public class Singleton {

    private volatile static Singleton uniqueInstance;

    private Singleton() {
    }

    public static Singleton getUniqueInstance() {
        if (uniqueInstance == null) {
            synchronized (Singleton.class) {
                if (uniqueInstance == null) {
                    uniqueInstance = new Singleton();
                }
            }
        }
        return uniqueInstance;
    }
}
```

注意点：

- 实例对象修饰符为 *volatile*
- 先检查对象是否存在，不存在的话再进入同步代码块
- 进入同步代码块后，再检查一次对象是否存在

## 总结

### 经典单例模式

优点：简单有效；缺点：多线程下可能会有问题。

### 同步 getInstance 方法

优点：解决了多线程下的问题；缺点：多次同步，性能不佳，产生浪费。

### 急切初始化

优点：简单、避免同步、多线程下正常工作；缺点：需要提前初始化，如果对象没有使用，会产生资源浪费。

### 双重检查加锁

优点：解决了多线程下不同步的问题，同时又避免了同步方法下的性能问题；缺点：代码有点多。


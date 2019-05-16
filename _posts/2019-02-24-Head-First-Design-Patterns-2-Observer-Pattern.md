---
layout: post
title: 《HeadFirst设计模式》学习日志（二）:观察者模式
date: 2019-02-24 14:23:00
author: admin
comments: true
categories: [Java]
tags: [Java,Design Patterns]

---



<!-- more -->

------

学习资料主要参考： 《Head First 设计模式》

------




* 目录
{:toc}

------

## 简述

观察者模式：让你的对象知悉现况。

观察者模式 = 主题（subject） + 观察者（observer）。

具体流程：

1. 注册（订阅）成为观察者
2. 等待通知
3. 主题有了新数据，所有观察者都会得到消息
4. 某个观察者退订，主题会将它从观察者列表中除名。

## 定义

> 定义了对象之间的一对多依赖，这样一来，当一个对象改变状态时，它的所有依赖者都会收到通知并自动更新。

```java
public interface Subject{
    void registerObserver(Observer ob);
    void removeObserver(Observer ob);
    notifyObservers();
}

public interface Observer{
    public void update(Data data);
}
```

观察者模式提供了一种对象设计，让主题和观察者之间松耦合，这样子它们依然可以交互，但是不太清楚批次的细节。

**设计原则：为了交互对象之间的松耦合设计而努力。**

## Java 内置的观察者模式

实现观察者接口（java.util.Observer），然后调用任何 Observable 对象的 addObserver() 方法。不想在当观察者时，调用 deleteObserver() 方法就可以了。

### 可观察者发出通知

Observable 对象发出通知时，先调用 setChanged 方法，标记状态已改变的事实，然后调用两种notifyObservers 方法的一个： notifyObservers() 或 notifyObservers(Object arg)。

### 观察者接收通知

```java
update(Observable o, Object arg)；
```

如果想“推”数据给观察者，就把数据作为对象传入 notifyObservers(Object arg) 中，否则，观察者就从可观察对象中“拉” 数据。

**不要依赖与观察者被通知的次序。**因为一旦观察者或可观察者的实现有所改变，通知次序就会改变。

### 缺点

Observable 是一个类，而不是一个接口。Java是不支持多继承的。




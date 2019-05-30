---
layout: post
title: Java 并发编程实战（零）： 总览与介绍
date: 2019-05-29 15:18:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

近日发现对 Java 的并发了解仍然不深入，特此找本书来学一下，中文名称叫做《Java 并发编程实践》，英文名叫做《Java Concurrency In Practice》。


<!-- more -->

* 目录
{:toc}
---


# 脑图

[![](/images/posts/JavaConcurrencyInPractice-Overview.png)](/images/posts/JavaConcurrencyInPractice-Overview.png)

可以看出这本书总共分为四大部分，分别为：

1. 基础
2. 构建并发应用、
3. 应用的活跃性、测试、性能
4. 并发的高级话题

# 介绍

## 1. 并发简史

背景：

1. cpu越来越多，不能让浪费，需要提高资源利用率。
2. 对于不同用户，需要对计算机上的资源有平等的使用权，即公平性
3. 有些事情分解起来做，更加简单，这是便利性

## 2. 优点

1. 发挥多处理器的强大能力
2. 建模的简单性
3. 异步事件的简化处理
4. 响应更灵敏的用户界面

## 3. 风险

### 安全性问题

常见的一个场景就是多个线程同时更新一个共有变量，可能会造成结果异常。

### 活跃性问题

安全性的含义是“永远不会发生糟糕的事情”，而活跃性的含义是“某件正确的事情最终会发生”。

常见的情况比如死锁、饥饿、活锁等。

### 性能问题

虽说多线程可以提高性能，但是有些情况下多线程的引入，也会造成性能问题。

因为线程的创建与销毁，线程间切换，都是有开销的。

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

创建状态依赖类的最简单的房就是在JDK提供了的状态依赖类基础上构造。例如第八章的ValueLactch，如果这些不满足，可以使用Java语言或者类库提供的底层机制来构造，包括：
- 内置的条件队列
- condition
- AQS

<!-- more -->

---

* 目录
{:toc}
---


## 未完待续。。。。。

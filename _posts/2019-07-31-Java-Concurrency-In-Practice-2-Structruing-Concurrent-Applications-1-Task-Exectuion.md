---
layout: post
title: Java 并发编程实战-学习日志（二）1：任务执行
date: 2019-07-31 20:13:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

这一章主要介绍 Java 中有用的并发模块。

大多数并发应用程序都是围绕“任务执行”来构造的：任务通常是一些抽象且离散的工作单元。

<!-- more -->

* 目录
{:toc}
---

# 在线程中执行任务

当围绕任务执行来设计应用程序时，第一步就是找出清晰的任务边界。

如果任务是相互独立的，那么有助于实现并发。

## 1. 串行地执行任务

在应用程序中，可以通过多种策略来调度任务，其中最简单的就是在单个线程中串行地执行各项任务。

```java
public class SingleThreadWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            Socket connection = socket.accept();
            handleRequest(connection);
        }
    }

    private static void handleRequest(Socket connection) {
        // request-handling logic here
    }
}
```

简单明了，但却性能低下，因为它每次只能处理一个请求。通常在处理过程中，I/O 操作是比较耗时的，这时候 CPU 其实处于空闲状态，很浪费资源。


## 2. 显示地为任务创建线程

```java
public class ThreadPerTaskWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            final Socket connection = socket.accept();
            Runnable task = new Runnable() {
                public void run() {
                    handleRequest(connection);
                }
            };
            new Thread(task).start();
        }
    }

    private static void handleRequest(Socket connection) {
        // request-handling logic here
    }
}
```

代码如上，但是千万不要这么做，因为可能会创建过多线程，使得程序崩溃。





### 未完待续。。。。。。


---
layout: post
title: 深入理解 Linux 内核（零）：总览及绪论
date: 2020-04-10 18:11:04
author: admin
comments: true
categories: [Linux]
tags: [Linux, Linux Kernel]
---

Linux 作为主流且好评如潮的操作系统 ，内核是什么样子的呢？来了解一下吧。

<!-- more -->

---

* 目录
{:toc}
---



## 全书脑图

[![](/images/posts/UnderstandingLinuxKernel-Overview.png)](/images/posts/UnderstandingLinuxKernel-Overview.png)



可以看出这本书大概分为十一个部分，涵盖了Linux内存管理、文件系统、进程管理、进程通信等。

来一步步学习吧。



## 绪论

Linux 是 Unix-like 操作系统大家庭中的一名成员。

### 1. 与其他 Unix 内核的比较

Linux 内核2.6版本遵循IEEE POSIX 标准。

与其他内核相比：

- 单块结构的内核
- 编译并运行的传统 Unix 内核
- 内核线程
- 多线程应用程序支持
- 抢占式内核
- 多处理器支持
- 文件系统
- STREAMS

优势：

- 免费
- 所有成分都可以定制
- 可以运行在便宜、低档的硬件平台
- 强大
- 开发者都很出色
- 内核非常小，而且紧凑
- 与许多通用操作系统高度兼容
- 很好的技术支持

### 2. 操作系统基本概念

操作系统是一系列程序的集合。在这个集合中，最重要的程序是内核。它在操作系统启动时，直接被装入RAM中。

操作系统必须完成两个主要目标：

- 与硬件部分互交，为包含在硬件平台上的所有底层可编程部件提供服务
- 为运行在计算机系统上的应用程序(用户程序)提供执行环境



## 未完待续。。。





## 参考资料

[深入理解LINUX内核(第三版)](https://book.douban.com/subject/2287506/)

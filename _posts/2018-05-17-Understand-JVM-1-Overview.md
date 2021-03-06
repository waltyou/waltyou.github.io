---
layout: post
title: 《深入理解Java虚拟机：JVM高级特性与最佳实践--第二版》学习日志（一）：概述
date: 2018-5-17 20:16:04
author: admin
comments: true
categories: [Java]
tags: [Java, JVM]
---

Java虚拟机（JVM）是java语言这么流行的基础。因为它，程序员代码与内存管理进行了一定程度上的隔离；因为它，java可以一次编译，多次运行；因为它，java可以跨越不同平台。

一定很好奇它是怎么工作的吧，来开始吧。

<!-- more -->
---

学习资料主要参考： 《深入理解Java虚拟机：JVM高级特性与最佳实践(第二版)》，作者：周志明

---



* 目录
{:toc}
---

# 全书脑图

[![](/images/posts/UnderstandJVM.png)](/images/posts/UnderstandJVM.png)



---

# 概述

全书分为五大部分。

第一部分简单介绍了一下java，回顾了历史，展望了未来。

第二部分和第三部分是全书重点。

第二部分讲述了java是如何进行自动的内存管理。
重点包括内存区域的介绍、垃圾回收机制的介绍、如何监控虚拟机和处理它故障，最后还有几个实际例子进行实践。

第三部分，主要是为了讲述虚拟机是如何加载并执行类的。首先介绍类文件的结构，接着讲如何加载类，再讲如何执行类，最后拿实际例子练手。

第四部分讲述了早期与晚期，两个时间段该如何对代码进行优化。

最后一部分讲述了从内存模型角度该如何看待并发，以及了解内存模型后，代码方面该怎么更高效、更方便。

---

# 走进Java

## 展望Java未来

### 1. 模块化

系统和平台越来越复杂，模块化是解决这个问题的一个重要途径。

### 2. 混合语言

有越来越多的语言运行在JVM上。当各种语言各取所长，而且又能很好的组合工作，那么对我们解决越来越复杂的需求时，绝对是个福音。

### 3. 多核并行

并行必不可少。

1.7中引入了fork/join框架。1.8中有lambda和stream。

还有OpenJDK的子项目Sumatra，它可以使用GPU的计算能力。

### 4. 进一步丰富语法

1.5扩充加入了：自动装箱、泛型、动态注解、枚举、可变长参数、遍历循环等。

### 5. 64位虚拟机

主流的CPU已经开始64位架构了，JVM也最终会发展至64位。

## 编译JDK


### OpenJDK、Sun/OracleJDK两者的不同点

这是自己曾经遇到过的一个面试题，特此总结一下。

1. OracleJDK采用了商业实现，OpenJDK使用的是开源的。
2. OracleJDK中存在一些商用闭源的功能：Java Fglight Recorder。OpenJDK有Font renderer。
3. 两者之间的代码大部分都一样。
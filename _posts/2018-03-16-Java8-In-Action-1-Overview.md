---
layout: post
title: 《Java 8 in Action》学习日志（一）：总览
date: 2018-3-15 22:15:04
author: admin
comments: true
categories: [Java]
tags: [Java,Java8]
---


Java 8 早已发布过去四年，但是发现自己对其新特性还不清楚，所以决定学习一下，顺便做个日志记录一下自己学习过程。

学习资料主要参考： 《Java 8 In Action》、《Java 8实战》，以及其源码：[Java8 In Action](https://github.com/java8/Java8InAction)

<!-- more -->
---



* 目录
{:toc}
---

# 全书脑图

[![](/images/posts/Java+8+In+Action.png){:width="400" height="300"}](/images/posts/Java+8+In+Action.png)

---

# 梳理脉络

通过脑图可以看出，全书分为四个部分：

1. 基础知识，重点是**为何关心java8**，**行为参数化**和**lambda**
2. 函数式编程，重点是全面系统的介绍**Stream**
3. Java8的其他改善点：
    1. **重构/测试/调试**
    2. **默认方法（Default Function）**
    3. **Optional替代null**
    4. **CompletableFuture 组合式异步编程**
    5. **日期时间API**
4. Java8之上：对**函数式编程**的思考，函数编程的技巧，与Scala的比较


接下来会先依次学习各个部分。


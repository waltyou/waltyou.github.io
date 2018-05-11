---
layout: post
title: Hadoop中的map和mapper的区别，reduce和reducer的区别
date: 2018-05-11 15:16:04
author: admin
comments: true
categories: [Hadoop]
tags: [Hadoop, Big Data, MapReduce]
---

Hadoop中的MapReduce中，有两个主要的步骤，一个是map，一个是reduce。

在任务运行时，我们又常说启动了多个mapper，多少个reducer。

那么map和mapper的区别，reduce和reducer到底有什么区别？该怎么区分它们呢？

<!-- more -->

---
# map与reduce

个人理解，可以代表三个意思：
1. 从MapReduce计算模型角度来讲，这两个词代表的是两个计算步骤。
2. 在MapReduce的任务调度中，这两个词又可以代指map task，reduce task。
3. 从代码层面中，它们是Mapper类中的map函数，Reducer类中的reduce函数

所以结合第2点和第3点来看，Reduce task其实是一个运行在node上，且执行Redcuer类中reduce函数的程序。可以把Reduce task当作是Reducer的一个实例。

同理可理解一下map task。


# Mapper 与 Reduce

1. 从代码层面中，Mapper和Reducer是两个类。
2. 在MapReduce的任务调度中， Mapper和Reducer分别是数据处理的第一阶段和第二阶段。
3. 还有一种理解是，mapper和reducer可以看作是一个计算资源的slot，它可以被用作完成许多map task或者reduce task。
它们可以被安排task，完成之后，又可以处理新的task。


---

# 参考链接

https://stackoverflow.com/questions/35861206/difference-between-reduce-task-and-a-reducer
http://data-flair.training/forums/topic/what-is-reducer-in-hadoop

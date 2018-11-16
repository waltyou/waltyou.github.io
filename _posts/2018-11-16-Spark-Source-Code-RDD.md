---
layout: post
title: Spark 源码学习之 RDD
date: 2018-11-16 15:50:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

读一读 Spark 的源码，既可以学习 Spark， 也可以学习 scala。

先来看一看最重要的基础类： RDD。

通篇主要参考 github 上 [spark 2.4.0](https://github.com/apache/spark/tree/v2.4.0)版本。

<!-- more -->

---

## 目录
{:.no_toc}

* 目录
{:toc}

---

# 总览

RDD 是英文版弹性分布式数据集（Resilient Distributed Dataset）的首字母缩写。

RDD 源码下总共分为三个部分：一个 class RDD ，两个 object，分别为 object RDD 与 object DeterministicLevel。

## class RDD

这是是学习的重点，简单介绍一下，下文有详细介绍。

它是一个抽象类，接收两个参数：
- _sc: SparkContext 
- deps: Seq[Dependency[_]]


## object RDD

主要是定义隐式函数，为特定类型的RDD提供额外的功能。

比如它之中有个名为 rddToPairRDDFunctions 的函数，它可以把一个 RDD 转换为一个 PairRDDFunctions（它是一个键值对 RDD）。
然后我们就可以使用 PairRDDFunctions 的 reduceByKey 方法了。

## object DeterministicLevel

它定义了三个枚举变量，来代表RDD输出的确定性级别（即`RDD＃compute`返回的内容）。 

这解释了Spark重新运行RDD任务时输出的差异。 

有3个确定性水平：
1. DETERMINATE 确定：重新运行后，RDD输出始终是相同顺序的相同数据集。
2. UNORDERED 无序：RDD输出始终是相同的数据集，但重新运行后顺序可能不同。
3.INDETERMINATE 不确定。 重新运行后，RDD输出可能会有所不同。

请注意，RDD的输出通常依赖于父RDD。 
当父RDD的输出是INDETERMINATE时，RDD的输出很可能也是INDETERMINATE。

---

# class RDD

class RDD 的代码主要分为三部分：
- 需要子类实现的方法与字段
- 适用与所有RDD的方法与字段
- 其他内部方法与字段

## 需要RDD子类实现的方法

在class RDD 的开头注释中，提到区分 RDD 的5个点：
- 一个 partitions 列表
- 一个可以计算所有split的函数
- 一个存储与其他RDD依赖关系的列表
- 可选的，对于 key-value RDDs，有一个 Partitioner 
- 可选的，一个列表存储着计算每个split的偏好地址（减少网络传输）

这个五条分别对于了子类需要实现的方法与变量。

### compute

    def compute(split: Partition, context: TaskContext): Iterator[T]

来计算一个给定 partition。

### getPartitions

    protected def getPartitions: Array[Partition]

返回这个 RDD partitions的集合。

这个方法只会被调用一次，所以它可以是个耗时的操作。

### getDependencies

    protected def getDependencies: Seq[Dependency[_]] = deps

返回这个 RDD 是如何依赖它的 parent RDD们的。

这个方法也只会调用一次。

### getPreferredLocations

    protected def getPreferredLocations(split: Partition): Seq[String] = Nil

可选方法，返回一个 partition 偏好的地址（就近原则）。

### partitioner

    @transient val partitioner: Option[Partitioner] = None

可选方法，来说明数据是如何分区的，举个例子：hash-partitioned



---

## 未完待续。。。。。。

---
layout: post
title: Mastering Apache Spark Core（五）：RDD
date: 2019-01-21 12:34:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark, Mastering Apache Spark]
---



<!-- more -->

------

## 目录
{:.no_toc}

* 目录
{:toc}


------

## 简述

使用RDD Spark隐藏数据分区和分布，从而允许他们使用更高级别的编程接口（API）为四种主流编程语言设计并行计算框架。

创建RDD的动机是当前计算框架处理效率低下的两种类型的应用程序：

1. 机器学习和图形计算中的迭代算法（**iterative algorithms**）。
2. 交互式数据挖掘工具作为同一数据集的即席查询（**interactive data mining tools**）。

目标是在多个数据密集型工作负载中重用中间内存结果，而无需通过网络复制大量数据。



## RDD的类型

- **ParallelCollectionRDD** - 是`SparkContext.parallelize` 和 `SparkContext.makeRDD`的结果
- **CoGroupedRDD** - 将一对父RDD合并为一个RDD。`RDD.cogroup(…)`.
- **HadoopRDD** 是一个提供了使用旧版MapReduce API读取存储在HDFS中数据的核心功能的RDD. 最值得注意的用例是作为`SparkContext.textFile`的返回RDD.
- **MapPartitionsRDD** - 进行操作后的结果，比如操作： `map`, `flatMap`, `filter`, `mapPartitions`, etc.
- **CoalescedRDD** - 重新分配或合并转换的结果
- **ShuffledRDD** - shuffled的结果
- **PipedRDD** - 由管道元素创建到分叉外部进程的RDD
- **PairRDD** (隐式地通过 `PairRDDFunctions`转换获得) - 是一个 key-value 对RDD。操作 `groupByKey` 和 `join` 会产生.
- **DoubleRDD** (隐式地通过 `org.apache.spark.rdd.DoubleRDDFunctions`) 是个 `Double` 类型的RDD
- **SequenceFileRDD** (隐式地通过 `org.apache.spark.rdd.SequenceFileRDDFunctions`) 是个可以被保存为 `SequenceFile` 的RDD.

## RDD Lineage

RDD Lineage（又名RDD运算符图或RDD依赖图）是RDD的所有父RDD的图。 它是将转换应用于RDD并创建逻辑执行计划的结果。

[![](/images/posts/rdd-lineage.png)](/images/posts/rdd-lineage.png)

上面的RDD图可能是以下一系列转换的结果：

```scala
val r00 = sc.parallelize(0 to 9)
val r01 = sc.parallelize(0 to 90 by 10)
val r10 = r00 cartesian r01
val r11 = r00.map(n => (n, n))
val r12 = r00 zip r01
val r13 = r01.keyBy(_ / 20)
val r20 = Seq(r11, r12, r13).foldLeft(r10)(_ union _)
```

**逻辑执行计划**从最早的RDD（不依赖于其他RDD或引用缓存数据的RDD）开始，以产生结果的action动作的RDD结束。

## Partitions

**partition** （又名 split）是大型分布式数据集的逻辑块。Spark使用 Partitions 来管理数据，这些 Partitions 有助于并行化分布式数据处理，只需最少的网络流量即可在执行程序。
默认情况下，Spark会尝试从靠近它的节点将数据读入RDD。由于Spark通常访问分布式分区数据，为了优化转换操作，它创建了用于保存数据块的分区。

默认情况下，为每个HDFS分区创建一个分区。

当一个阶段执行时，您可以在Spark UI中看到给定阶段的分区数。

可以使用 `rdd.partitions.size` 方法来查看 rdd 的分区数目。

每当发生 shuffle 时，分区就会在节点之间重新分配。

初始化 rdd 的时候，也可以指定分区数目：`sc.textFile(path, partition)`，但是它只能用于未压缩的数据。面对压缩数据，我们需要在读取完成之后，进行重分区。

```scala
repartition(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T]
```

以上方法会触发 shuffle 操作。

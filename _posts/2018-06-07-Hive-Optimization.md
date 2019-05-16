---
layout: post
title: Hive 调优
date: 2018-06-07 15:16:04
author: admin
comments: true
categories: [Hive]
tags: [Big Data, Hive]
---

当数据量变得越来越大，或者处理逻辑变得越来越复杂时，如何优化hive的执行就显得越发重要。
这里有7种优化方式，来依次介绍一下。

<!-- more -->
---



* 目录
{:toc}
---

## Tez 执行引擎

Tez 是一个基于hadoop yarn的应用框架。

它可以执行一般数据处理任务所产生的复杂指向、非循环图。

此外，为了在Hadoop上编写本地YARN应用程序，以弥合交互式和批量工作负载的范围，Tez为开发人员提供了一个API框架。

了解更多：[Apache TEZ](https://tez.apache.org/)

## 使用合适的文件格式

ORCFILE 就是一个合适的文件格式。它的全名是Optimized Row Columnar (ORC) file。

据[官方文档](https://orc.apache.org/)介绍，这种文件格式可以提供一种高效的方法来存储Hive数据。它的设计目标是来克服Hive其他格式的缺陷。

运用ORC File可以提高Hive的读、写以及处理数据的性能。

更具体地说，ORC将原始数据的大小减小到75％。

## Partitioning

通过Partitioning将数据集的各个列的所有条目分离并存储在它们各自的分区中，这样子就可以只对某些我们想要的分区下数据进行处理，而不是处理整个数据集。

## Bucketing

经过Partition之后，数据集有可能还是很大。这时就需要引入bucket了，它允许用户将表格数据集划分为更易管理的部分。

## Vectorization

通过Vectorization， hive一次可以按照1024行的批处理来执行，而不是每次执行一行。

使用vectorized：
```
set hive.vectorized.execution = true
set hive.vectorized.execution.enabled = true
```

## 基于成本的优化（CBO）

CBO在Hive中增加了基于查询成本的进一步优化。这会导致潜在的不同决策：如何排序连接，要执行的连接类型，并行度等。

```
set hive.cbo.enable=true;
set hive.compute.query.using.stats=true;
set hive.stats.fetch.column.stats=true;
set hive.stats.fetch.partition.stats=true;
```

然后，通过运行Hive的“analyze”命令来为CBO准备数据，以收集我们想要使用CBO的表的各种统计信息。

## Hive 索引

对于原始表使用索引会创建一个单独的被称为索引表的reference。

当我们在一个有索引的表上执行查询时，查询不需要扫描表中的所有行，这就是使用索引的主要优点。有了索引之后，hive首先检查索引，然后转到特定列并执行操作。

因此，维护索引将更容易使Hive查询数据，然后在较少的时间内执行所需的操作。

这里有更详细的介绍：[Apache Hive View and Hive Index](https://data-flair.training/blogs/hive-view-hive-index/)

---

# 参考链接
1. [7 Best Hive Optimization Techniques](https://data-flair.training/blogs/hive-optimization-techniques/)

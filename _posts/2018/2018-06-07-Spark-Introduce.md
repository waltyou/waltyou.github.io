---
layout: post
title: Spark 介绍
date: 2018-06-07 16:16:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark]
---

继Hadoop之后，又一项大数据处理利器：Spark出世。来了解一下它。

<!-- more -->
---



* 目录
{:toc}
---

# Spark的基本介绍

## 1. 什么是Spark?

简单说，它是个开源的大数据处理引擎。它有许多API，可以更好的帮助数据开发者对数据进行Streaming，机器学习或SQL等操作。

同时，它可以与大数据生态圈良好的整合。它可以访问Hadoop数据源，也可以在Hadoop集群上运行。

与Hadoop不同的是，它可以基于内存进行迭代运算。

我们可以用Java、Scala、Python和R来进行编程。

## 2. Spark 的优缺点

### 1）优点

1. 高数据处理速度
2. 天然的动态性
3. Spark中的内存计算
4. 可重用性
5. 容错
6. 实时流处理
7. 懒惰评估
8. 支持多种语言：Java，R，Scala，Python
9. 支持复杂的分析
10. 与Hadoop集成
11. Spark GraphX 支持图形和图形并行计算
12. 成本效益

### 2）缺点

1. 不支持实时处理
2. 小文件问题
3. 没有文件管理系统
4. 成本非常昂贵
5. 算法的数量较少
6. 手动优化
7. 迭代处理
8. 延迟

## 3. Spark的使用场景

- 金融业：它有助于访问和分析银行部门的许多参数，例如电子邮件，社交媒体档案，通话录音，论坛等等
- 电子商务行业： 有助于获得有关实时交易的信息。而且，这些被传递给流聚类算法。
- 媒体和娱乐业：从实时的游戏事件中，识别模式
- 旅游业：可以帮助用户通过加快个性化建议来规划一次完美的旅程

---

# Spark 的组件

## 1. Spark Core

Spark Core是Spark的中心点。基本上，它为所有的Spark应用程序提供了一个执行平台。此外，为了支持广泛的应用程序，Spark提供了一个通用平台。

## 2. Spark SQL

在Spark的顶部，Spark SQL使用户能够运行SQL / HQL查询。

我们可以使用Spark SQL处理结构化以及半结构化数据。

此外，它还可以在现有部署中将未修改的查询运行速度提高100倍。

## 3. Spark Streaming

基本上，在实时流媒体中，Spark Streaming支持强大的交互式和数据分析应用程序。此外，直播流将转换为可以在Spark Core顶部执行的微批次。

## 4. Spark MLlib

MLlib 既提供了高效率、高质量的机器学习算法。此外，它是数据科学家最热门的选择。由于它能够进行内存数据处理，因此可以大大提高迭代算法的性能。

## 5. Spark GraphX

Spark GraphX基本上是构建在Apache Spark之上的图形计算引擎，可以按比例处理图形数据。

## 6. SparkR

它是R包，提供轻量级的前端。而且，它允许数据科学家分析大型数据集，还允许从R shell交互式地运行作业。

SparkR背后的主要思想是探索不同的技术来将R的可用性与Spark的可扩展性结合起来。

---

# 参考链接
1. [Spark Tutorial – Learn Spark Programming](https://data-flair.training/blogs/spark-tutorial/)

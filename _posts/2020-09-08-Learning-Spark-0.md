---
layout: post
title: Learning Spark (0) - 总览
date: 2020-09-08 18:11:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark, Learning Spark]
---

Databricks 出了本《Learning Spark》的电子书，来学习一下。 

<!-- more -->

---

* 目录
{:toc}
---

## 总览

[![](/images/posts/LearningSpark.png)](/images/posts/LearningSpark.png)

可以看出这本书总共分为五大部分，分别为：

1. 基本介绍
2. Spark SQL 使用与调优
3. Streaming
4. MLlib
5. Spark 3.0 的新特性


电子书下载地址： https://databricks.com/p/ebook/learning-spark-from-oreilly



## Structured APIs

### 1. RDD 到底是什么？

RDD 是 Spark 最基础的抽象。它分为三部分：

1. Dependencies 依赖
2. Partitions 分区
3. Compute funtion: Partitions => Iterator[T] 计算函数

第一部分：依赖。定义了这个RDD是从何处来的，也保证了必要时重新计算这个RDD。第二部分：partitions 分区。将数据分区存储，天然支持并行计算。第三部分计算函数可以将消费分区数据，然后得到我们想要的 Iterator[T].

整体架构简洁明了。但是也存在几个问题。其中一个就是compute function对Spark是不透明的。对于Spark来讲，compute function 只是一个 lambda expression，不管它实际上是join、filter、select。 另外一个问题是 Iterator[T]  的数据类型，对Python RDD来讲，也是透明的，Spark只知道它是一个python的 generic object。

而且，因为无法检查计算函数，那就无从谈起优化这个函数。另外因为不知道数据类型，也无法直接获取某种类型的某列，Spark 只能将它序列化为字节，而且不能使用任何压缩技术。

毫无疑问，这些不透明性，影响了Spark优化我们的计算，那么解决方案是什么呢？





## 未完待续。。。










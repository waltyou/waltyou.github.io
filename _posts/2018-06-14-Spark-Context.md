---
layout: post
title: SparkContext 学习
date: 2018-06-13 16:07:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark]
---

SparkContext是Apache Spark功能的入口，任何Spark驱动程序应用程序最重要的步骤都是来生成SparkContext，来详细了解一下它。

<!-- more -->
---



* 目录
{:toc}
---

# 简介

## 1. 能做什么

它允许Spark应用程序在资源管理器的帮助下访问Spark Cluster。资源管理器可以是以下三种中的一种：Spark Standalone，YARN和Apache Mesos。

[![](/images/posts/SparkContext-Apache-Spark.jpg)](/images/posts/SparkContext-Apache-Spark.jpg)

## 2. 创建

要创建SparkContext，首先应该创建SparkConf。 SparkConf有一个配置参数，Spark驱动程序（Driver）会将这些参数，传递给SparkContext。

这些参数，指导了spark如何在集群上去请求资源，同时也控制了每个work node的上的container的数目、内存大小和core的数量。

## 3. 具体使用

在创建 SparkContext 对象之后，就可以通过它来调用spark函数，比如textFile, sequenceFile, parallelize等。

同时它就可以用来创建RDD，广播变量和作为累加器，进入Spark服务和运行作业。

所有这些都可以在SparkContext停止之前执行。

## 4. 停止 SparkContext

直接调用sparkcontext的stop方法即可。

---

# WordCount例子

```scala
package org.apache.spark.examples

import org.apache.spark.{SparkConf, SparkContext}

/**
  * Created by waltyou on 18-6-14.
  */
object WordCount {

  def main(args: Array[String]) {
    //Create conf object
    val conf = new SparkConf()
      .setAppName("WordCount")

    //create spark context object
    val sc = new SparkContext(conf)
    //Check whether sufficient params are supplied
    if (args.length < 2) {
      println("Usage: ScalaWordCount <input> <output>")
      System.exit(1)
    }
    //Read file and create RDD
    val rawData = sc.textFile(args(0))

    //convert the lines into words using flatMap operation
    val words = rawData.flatMap(line => line.split(" "))

    //count the individual words using map and reduceByKey operation
    val wordCount = words.map(word => (word, 1)).reduceByKey(_ + _)

    //Save the result
    wordCount.saveAsTextFile(args(1))
    //stop the spark context
    sc.stop

  }

}

```

记得在Run Configuration里设置好 spark.master，以及输入文件路径和输出目录即可。

---

# SparkContext 在Spark中的功能

## 1. 获取Spark应用程序的当前状态

### SpkEnv

它是Spark公共服务的运行时环境。它与其他部件互相交互，为Spark应用程序建立分布式计算平台。

它为正在运行的spark 应用保留其所需要的服务，服务其中包括记录驱动程序和执行程序的不同环境，从而可以通过SpkEnv来标识 Spark的运行环境。

### SparkConf

SparkConf 可以设置spark能处理的最大应用程序数量，并可针对每个应用程序分别进行个性化配置。

它的使用也很简单，一些常见的属性，如主URL和应用程序名称，都可以通过set（）方法来配置任意的键、值对。

### 部署环境（即master URL）

Spark部署环境有两种类型，即本地和集群。

本地模式是非分布式单JVM部署模式。所有执行组件（driver，executor，LocalSchedulerBackend和master）都存在于同一个JVM中。
这是唯一一个driver可以对执行起作用的模式。
为了测试，调试或演示目的，本地模式是合适的，因为它不需要提前设置来启动火花应用程序。

在集群模式下，Spark以分布模式运行。

## 2. 设置配置

### Master URL

master方法返回部署环境正在使用的spark.master当前值。

### 本地属性 - 创建逻辑作业组

本地属性概念的目的是通过属性创建逻辑作业组，这些属性使从不同线程启动的独立作业可以属于单个逻辑组。

你可以设置一个本地属性，用来影响从线程提交的Spark作业。

### 默认日志记录级别

它允许您在Spark应用程序中设置根登录级别，例如Spark Shell

## 3. 访问各种服务

它还有助于访问TaskScheduler，LiveListenBus，BlockManager，SchedulerBackend，ShuffelManager和可选的ContextCleaner等服务。

## 4. 取消 Job

cancleJob只是简单地要求DAGScheduler放弃一个Spark作业。

## 5. 取消 stage

cancleStage只是简单地要求DAGScheduler放弃一个Spark stage。

## 6. 闭包清理

每次发生Action操作时，Spark都会清理关闭操作，Action的主体会在清理前，序列化并通过网络发送、执行。

这个操作是 SparkContext的 clean方法来完成的，这个方法也被叫做 ClosureClean.clean。

它不仅清理闭包，而且也清理引用闭包。

## 7. 注册Spark监听器

我们可以通过 addSparkListener 方法，注册一个自定义的 SparkListenerInterface。或者通过 spark.extraListeners 设置，来注册也是可以的。

## 8. 可编程动态分配

它还提供以下方法作为开发者API，用于动态分配executor：requestExecutors，killExecutors，requestTotalExecutors，getExecutorIds。

## 9. 访问持久性RDD

getPersistentRDDs给出了已通过缓存标记为持久的RDD的集合。

## 10. unpersist RDD

从主模块管理器和内部persistentRdds映射中，unpersist删除RDD。


---

# 参考链接
1. [Learn SparkContext – Introduction and Functions](https://data-flair.training/blogs/learn-apache-spark-sparkcontext/)

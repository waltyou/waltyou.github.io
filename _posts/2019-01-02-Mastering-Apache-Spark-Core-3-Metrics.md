---
layout: post
title: Mastering Apache Spark Core（三）：Spark Metrics
date: 2019-01-02 14:34:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark, Mastering Apache Spark]

---

Spark Metrics 提供了 Spark 子系统（也称为度量实例）的执行指标，例如： Spark应用程序的驱动程序或Spark Standalone集群的主服务器。

<!-- more -->

------

## 目录
{:.no_toc}

* 目录
{:toc}

------

## 总览

是什么：

- 是一个Java库，可以让您无比深入地了解代码在生产中的作用。
- 提供了一个强大的工具包，用于衡量生产环境中关键组件的行为。

## MetricsSystem

Spark子系统的度量标准源（sources）和汇点（sinks）注册表。

MetricsSystem 使用 Dropwizard Metrics的MetricRegistry作为Spark和度量库之间的集成点。

Spark子系统可以通过 SparkEnv.metricsSystem 属性访问MetricsSystem。

```scala
val metricsSystem = SparkEnv.get.metricsSystem
```

spark-metrics-MetricsSystem-driver.png

[![](/images/posts/spark-metrics-MetricsSystem-driver.png)](/images/posts/spark-metrics-MetricsSystem-driver.png)

MetricsSystem最多可以有一个MetricsServlet JSON指标接收器（默认情况下已注册）。

创建时，MetricsSystem请求MetricsConfig初始化。

[![](/images/posts/spark-metrics-MetricsSystem.png)](/images/posts/spark-metrics-MetricsSystem.png)

### 注册度量标准源

```scala
registerSource(source: Source): Unit
```

registerSource将源添加到源内部注册表。

registerSource为metrics源创建一个标识符，并将其注册到MetricRegistry。

### Source

```scala
package org.apache.spark.metrics.source

trait Source {
  def sourceName: String
  def metricRegistry: MetricRegistry
}
```

### Sink

```scala
package org.apache.spark.metrics.sink

trait Sink {
  def start(): Unit
  def stop(): Unit
  def report(): Unit
}
```



## MetricsConfig

度量标准系统配置。

**metrics.properties** 是默认度量标准配置文件。它使用 spark.metrics.conf 配置属性进行配置。在使用Spark的CLASSPATH之前，首先直接从路径加载文件。

MetricsConfig还接受使用spark.metrics.conf.-prefixed配置属性的度量配置。

Spark附带了conf / metrics.properties.template文件，它是度量配置的模板。
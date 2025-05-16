---
layout: post
title: 精通 Apache Spark 源码 | 问题 01・一、核心架构 | SparkContext 初始化链路（DAGScheduler/TaskScheduler 与集群管理器交互）
date: 2025-05-16 13:39:04
author: admin
comments: true
categories: [Spark]
tags: [BigData, Spark]
---

<!-- more -->

---

* 目录
{:toc}
---

## 前言

这是[本系列](../master-in-apache-spark-with-source-code-00)的第一个问题，属于第一大类“核心架构”，问题描述为：

```markdown
**Trace SparkContext initialization**: What key components (DAGScheduler, TaskScheduler, SchedulerBackend) are created, and how do they interact with cluster managers (YARN/Kubernetes/Standalone)?  
   *Key files:* `SparkContext.scala`, `SparkSession.scala`
```

## 解答

TODO

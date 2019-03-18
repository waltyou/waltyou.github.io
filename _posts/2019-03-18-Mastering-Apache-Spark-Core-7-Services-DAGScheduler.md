---
layout: post
title: Mastering Apache Spark Core（七）：核心服务 DAGScheduler
date: 2019-03-18 14:01:04
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

### 简介

**DAGScheduler** 是Apache Spark的调度层，它实现了面向阶段的调度（**stage-oriented scheduling**）。它将逻辑执行计划（**logical execution plan**）（使用RDD转换构建的依赖关系的RDD谱系）转换为物理执行计划（**physical execution plan**）（使用 stages）。

[![](/images/posts/dagscheduler-rdd-lineage-stage-dag.png)](/images/posts/dagscheduler-rdd-lineage-stage-dag.png)

调用一个 action 后，SparkContext 将逻辑计划移交给 `DAGScheduler`，然后转换为一组作为 TaskSet 进行执行的 stages 。

[![](/images/posts/dagscheduler-rdd-partitions-job-resultstage.png)](/images/posts/dagscheduler-rdd-partitions-job-resultstage.png)

DAGScheduler 的基本概念是 jobs 和 stages，他们分别通过内部注册表和计数器跟踪。

DAGScheduler 仅在 dirver 中使用，是作为SparkContext初始化的一部分创建的（在TaskScheduler和SchedulerBackend准备好之后）。

[![](/images/posts/dagscheduler-new-instance.png)](/images/posts/dagscheduler-new-instance.png)

DAGScheduler 在Spark中做了三件事（详细解释如下）：

- 计算出一个执行的 DAG
- 确定运行每个任务的首选位置
- 处理由于shuffle输出文件丢失而导致的故障

DAGScheduler 为每个作业计算阶段的有向非循环图（DAG），跟踪实现哪些RDD和阶段输出，并找到运行作业的最小计划（即最优计划）。然后它将阶段提交给TaskScheduler。

除了提出执行 DAG 之外，DAGScheduler还根据当前缓存状态确定运行每个任务的首选位置，并将信息传递给TaskScheduler。

DAGScheduler 跟踪哪些RDD被缓存（或持久化）以避免“重新计算”它们，即重做shuffle的map侧。DAGScheduler 记得哪些 ShuffleMapStages 已经生成了输出文件（存储在BlockManagers中）。

DAGScheduler 只对RDD的每个分区的缓存位置坐标（即主机和执行程序ID）感兴趣。





## 未完待续。。。

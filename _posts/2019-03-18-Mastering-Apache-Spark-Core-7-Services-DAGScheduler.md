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

## 简介

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

此外，它处理由于shuffle输出文件丢失而导致的故障，在这种情况下可能需要重新提交旧阶段。不是由 shuffle 文件丢失而引起的 stage 内的故障，由TaskScheduler本身处理，它将在取消整个 stage 之前少次地重试每个任务。

DAGScheduler 使用事件队列体系结构（**event queue architecture**），一个线程可以发布 DAGSchedulerEvent 事件，例如提交的新作业或阶段，DAGScheduler 按顺序读取事件和执行它。

DAGScheduler按拓扑顺序运行。

DAGScheduler 使用 SparkContext，TaskScheduler，LiveListenerBus，MapOutputTracker 和 BlockManager作为其服务。但是，DAGScheduler 只接受 SparkContext（并为其他服务请求SparkContext）。

DAGScheduler 报告有关其执行的指标。

当 DAGScheduler 通过在RDD上执行 action 或直接调用 SparkContext.runJob（）方法来调度作业时，它会生成并行任务以计算每个分区的（部分）结果。

## 主要方法

### submitJob

提交作业。

```scala
submitJob[T, U](
  rdd: RDD[T],
  func: (TaskContext, Iterator[T]) => U,
  partitions: Seq[Int],
  callSite: CallSite,
  resultHandler: (Int, U) => Unit,
  properties: Properties): JobWaiter[U]
```

submitJob创建一个JobWaiter并发布一个JobSubmitted事件。

[![](/images/posts/dagscheduler-submitjob.png)](/images/posts/dagscheduler-submitjob.png)

在内部，submitJob执行以下操作：

1. 检查 `partitions` 是否引用输入rdd的可用分区
2. 增加 nextJobId 内部工作计数器
3. 当 `partitions` 数量为零时，返回 0-task JobWaiter
4. 提交 `JobSubmitted` 事件并返回 `JobWaiter`

当输入 `partitions` 引用不在输入rdd中的分区时，您可能会看到抛出 IllegalArgumentException：

```shell
Attempting to access a non-existent partition: [p]. Total number of partitions: [maxPartitions]
```

### submitMapStage

提交 ShuffleDependency 给 Execution。

```scala
submitMapStage[K, V, C](
  dependency: ShuffleDependency[K, V, C],
  callback: MapOutputStatistics => Unit,
  callSite: CallSite,
  properties: Properties): JobWaiter[MapOutputStatistics]
```

submitMapStage 创建一个JobWaiter（最终返回）并将 MapStageSubmitted 事件发布到  DAGScheduler Event Bus。

在内部，submitMapStage递增nextJobId内部计数器以获取作业ID。

然后，submitMapStage创建一个JobWaiter（具有作业ID和一个人造任务，但只有在整个阶段结束时才会完成）。

submitMapStage 宣布 application-wide 内的map stage提交（通过将 MapStageSubmitted 发布到 LiveListenerBus）。

如果要计算的分区数为0，则submitMapStage会抛出SparkException：

```shell
Can't run submitMapStage on RDD with 0 partitions
```

### runJob

运行作业。

```scala
runJob[T, U](
  rdd: RDD[T],
  func: (TaskContext, Iterator[T]) => U,
  partitions: Seq[Int],
  callSite: CallSite,
  resultHandler: (Int, U) => Unit,
  properties: Properties): Unit
```

runJob 向 DAGScheduler 提交一个动作作业并等待结果。

在内部，runJob 执行 submitJob，然后等待直到使用 JobWaiter 得到结果。



## 未完待续。。。

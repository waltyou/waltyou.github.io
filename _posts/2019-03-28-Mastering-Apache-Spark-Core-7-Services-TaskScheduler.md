---
layout: post
title: Mastering Apache Spark Core（七）： 核心服务 TaskScheduler
date: 2019-03-28 18:45:04
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

TaskScheduler 负责提交在Spark应用程序中执行的任务（根据调度策略）。

[![](/images/posts/sparkstandalone-sparkcontext-taskscheduler-schedulerbackend.png)](/images/posts/sparkstandalone-sparkcontext-taskscheduler-schedulerbackend.png)

TaskScheduler 使用`executorHeartbeatReceived`和`executorLost`方法跟踪Spark应用程序中的执行程序，这些方法分别用于通知活动和丢失的执行程序。

Spark附带以下自定义 TaskSchedulers：

- TaskSchedulerImpl - 默认的TaskScheduler（以下两个特定于YARN的TaskSchedulers扩展）。
- 在客户端部署模式下，YARN上的Spark的YarnScheduler。
- 在集群部署模式下，YARN上的Spark的YarnClusterScheduler。

## TaskScheduler的生命周期

在创建SparkContext时创建 TaskScheduler（通过为给定的主URL和部署模式调用SparkContext.createTaskScheduler）。

在SparkContext的生命周期中，内部 _taskScheduler 指向 TaskScheduler（并通过向HeartbeatReceiver RPC端点发送阻塞的TaskSchedulerIsSet消息来“通知”）。

在阻塞TaskSchedulerIsSet消息收到响应后，立即启动TaskScheduler。

application ID 和 application’s attempt ID 就是此时设定的（SparkContext使用应用程序ID设置spark.app.id Spark属性，并配置SparkUI和BlockManager）。

在SparkContext完全初始化之前，调用 TaskScheduler.postStartHook。

当SparkContext被停止时，内部_taskScheduler被清除（即设置为null）。

在DAGScheduler停止时，TaskScheduler停止。

## Task

Task（也叫 *command*）是为计算RDD分区而启动的最小的单独执行单元。

[![](/images/posts/spark-rdd-partitions-job-stage-tasks.png)](/images/posts/spark-rdd-partitions-job-stage-tasks.png)

一个 task， 拥有一个 runTask 方法和一个可选的偏好引用来选择正确的 executor。

Task 有两个具体实现：

- ShuffleMapTask，执行任务并将任务的输出分成多个桶（基于任务的分区）。
- ResultTask 执行任务并将任务的输出发送回驱动程序应用程序。

Spark作业的最后一个 stage 包含多个ResultTasks，而早期 stage 只能是ShuffleMapTasks。

Task 在执行程序上启动，并在 TaskRunner 启动时运行。

用更技术性的话来说，Task 是对某个 job 中的某个 stage 中的某个 RDD 的某个分区中的记录进行计算。

任务只能属于一个阶段并在单个分区上运行。阶段中的所有任务必须在后续阶段开始之前完成。

每个 stage 和 partition 都会逐个生成任务。

### run 方法

```scala
run(
  taskAttemptId: Long,
  attemptNumber: Int,
  metricsSystem: MetricsSystem): T
```

run 使用本地 BlockManager 注册任务（标识为 taskAttemptId ）。

运行创建一个TaskContextImpl，然后让它成为本任务的TaskContext。

运行检查_killed标志，如果启用，则终止任务（禁用interruptThread标志）。

run创建一个Hadoop CallerContext并设置它。

run运行任务。（这是执行自定义任务的runTask的时刻。）

最后，运行通知TaskContextImpl任务已完成（无论最终结果如何 - 成功或失败）。

如果出现任何异常，run 会通知TaskContextImpl任务失败。run 请求MemoryStore释放此任务的展开内存（适用于ON_HEAP和OFF_HEAP内存模式）。

run 请求MemoryManager通知任何等待执行内存的任务被释放唤醒并尝试再次获取内存。

run 取消设置任务的TaskContext。

### Task 状态：

- LAUNCHING
- RUNNING
- FINISHED
- FAILED
- KILLED
- LOST







## 未完待续。。。。
---
layout: post
title: Mastering Apache Spark Core（七）： 核心服务 DAGSchedulerEventProcessLoop
date: 2019-03-26 18:01:04
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

`DAGSchedulerEventProcessLoop`（**dag-scheduler-event-loop**）是一个`EventLoop` 中处理 `DAGSchedulerEvent` 事件的单个“业务逻辑”线程。

`DAGSchedulerEventProcessLoop`的目的是让一个单独的线程以异步和串行方式处理事件，即逐个处理事件，让DAGScheduler在主线程上完成它的工作。

## DAGSchedulerEvent 类别

### AllJobsCancelled

要求 DAGScheduler 取消所有正在运行或正在等待的工作。

### BeginEvent

TaskSetManager 通知 DAGScheduler 任务正在启动（通过taskStarted）。

### CompletionEvent

发布通知 DAGScheduler 任务已完成（成功与否）。

`CompletionEvent` 传达以下信息：

1. 完成 Task（task 字段）
2. TaskEndReason（reason 字段）
3. task 的结果 （result 字段）
4. Accumulator 更新
5. TaskInfo

### ExecutorAdded

DAGScheduler 被告知（通过executorAdded）一个 executor 在主机上运行了起来。

### ExecutorLost

发布通知 DAGScheduler 一个 executor 丢失了。

ExecutorLost传达以下信息：

1. execId
2. ExecutorLossReason

注意：当DAGScheduler被告知任务因FetchFailed异常而失败时，也会调用handleExecutorLost。

### GettingResultEvent

TaskSetManager 通知 DAGScheduler（通过taskGettingResult）任务已完成并且远程获取结果。

### JobCancelled

要求DAGScheduler取消工作。

### JobGroupCancelled

要求DAGScheduler取消工作组。

### JobSubmitted

在请求DAGScheduler提交作业或运行approximate job时发布。

JobSubmitted传达以下信息：

1. jobId
2. finalRDD
3. `func: (TaskContext, Iterator[_]) ⇒ _`
4. 需要计算的 partitions
5. 一个 `CallSite`
6. JobListener通知阶段的状态
7. 执行的属性

### MapStageSubmitted

发布通知 DAGScheduler，SparkContext提交了一个MapStage来执行（通过submitMapStage）。

MapStageSubmitted传达以下信息：

1. jobId
2. `ShuffleDependency`
3. 一个 `CallSite`
4. JobListener通知阶段的状态
5. 执行的属性



### ResubmitFailedStages

DAGScheduler被告知由于FetchFailed异常导致任务失败。

### StageCancelled

要求DAGScheduler取消一个阶段。

### TaskSetFailed

要求DAGScheduler取消TaskSet。






## 未完待续。。。





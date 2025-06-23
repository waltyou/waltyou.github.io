---
layout: post
title: 精通 Apache Spark 源码 | 问题 02・一、核心架构 | DAG 调度器作业划分逻辑（Stage 边界与 Shuffle 依赖解析）
date: 2025-06-23 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第二个问题，属于第一大类"核心架构"，问题描述为：

```markdown
**Spark's execution model**: How does the DAG scheduler split jobs into stages using shuffle dependencies? Explain stage boundary determination.  
   *Key files:* `DAGScheduler.scala`, `ShuffleDependency.scala`
```

## 解答

DAGScheduler 通过分析 RDD 之间的依赖关系，将一个 Job 拆分为多个 Stage。其核心依据是 **ShuffleDependency**（宽依赖）。

### 原理说明

- **NarrowDependency（窄依赖）**：如 map、filter、union 等操作，子 RDD 的每个分区只依赖父 RDD 的少量分区，可以流水线执行，不需要 stage 边界。
- **ShuffleDependency（宽依赖）**：如 groupByKey、reduceByKey 等操作，子 RDD 的每个分区依赖父 RDD 的所有分区，需要全局洗牌（shuffle），此处必须切分 stage。

**Stage 边界判定流程：**

1. DAGScheduler 从最终 RDD（action）向上递归遍历依赖链。
2. 每遇到一个 ShuffleDependency，就以此为界，将前后的 RDD 分别划分到不同的 Stage。
3. 没有 ShuffleDependency 的连续 RDD 形成一个 Stage（通常是 ResultStage）。
4. 每个 ShuffleDependency 会生成一个 ShuffleMapStage，负责输出 shuffle 文件，供下游 Stage 读取。

### 举例

比如来看看 `rdd.map(...).filter(...).reduceByKey(...)`， map/filter 是窄依赖，reduceByKey 是宽依赖，reduceByKey 前的操作属于一个 Stage，reduceByKey 之后属于另一个 Stage。

`reduceByKey` 在 PairRDDFunctions 中定义， 实际调用 `combineByKeyWithClassTag`, 它最后返回一个 ShuffledRDD。ShuffleRDD 的 `getDependencies` func 会返回一个 List[ShuffleDependency].

然后来看 DAGScheduler 的代码，DAGScheduler 会在 submitJob 后会生成一个 JobSubmitted 事件，然后 handleJobSubmitted 会处理它。

```scala
finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite)
```

createResultStage：

1. val (shuffleDeps, resourceProfiles) = getShuffleDependenciesAndResourceProfiles(rdd)：获取 RDD 的依赖链条上所有 shuffleDeps，相对于拿到了所有的stage分割点
2. val parents = getOrCreateParentStages(shuffleDeps, jobId)：根据所有的 shuffleDeps 创建所有的 parents stages， 它们的类型都是 ShuffleMapStage。
3. getOrCreateParentStages 循环调用 getOrCreateShuffleMapStage，getOrCreateShuffleMapStage 会先通过 shuffleIdToMapStage 判断 shuffleDep 是否在已经存在，如果存在直接返回，如果不存在，就创建新的。另外在创建新的之前，会补齐当前 shuffleDep 依赖链上所有的 ShuffleMapStage，并把它们放进 shuffleIdToMapStage，这样可以确保DAG中所有shuffle依赖都被补齐，避免遗漏。
4. 构建 stage id 和 ResultStage


---
layout: post
title: 精通 Apache Spark 源码 | 问题 02・一、核心架构 | DAG 调度器作业划分逻辑（Stage 边界与 Shuffle 依赖解析）
date: 2025-05-17 13:39:04
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

TODO
---
layout: post
title: 精通 Apache Spark 源码 | 问题 21・十一、高级优化 | 自适应查询执行（AQE 基于 Shuffle 统计的优化）
date: 2025-06-05 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第二十一个问题，属于第十一大类"高级优化"，问题描述为：

```markdown
**Adaptive Query Execution**: How does AQE optimize runtime plan using shuffle statistics (Spark 3+)?  
     *Key classes:* `AdaptiveSparkPlanExec`, `QueryStage`
```

## 解答

TODO
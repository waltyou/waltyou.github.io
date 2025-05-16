---
layout: post
title: 精通 Apache Spark 源码 | 问题 24・十二、调试工具 | Spark UI 指标采集（LiveListenerBus 数据流转分析）
date: 2025-06-08 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第二十四个问题，属于第十二大类"调试与可观测性"，问题描述为：

```markdown
**Spark UI architecture**: How do listeners (e.g., LiveListenerBus) collect metrics for web UI?  
     *Key files:* `SparkUI.scala`, `LiveEntity.scala`
```

## 解答

TODO
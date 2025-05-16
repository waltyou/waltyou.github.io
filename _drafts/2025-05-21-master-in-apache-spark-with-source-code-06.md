---
layout: post
title: 精通 Apache Spark 源码 | 问题 06・三、调度执行 | 本地化调度策略（进程/节点/机架本地任务优先级）
date: 2025-05-21 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第六个问题，属于第三大类"调度与执行"，问题描述为：

```markdown
**Locality-aware scheduling**: How does Spark prioritize process-local > node-local > rack-local tasks?  
   *Key logic:* `TaskSetManager.scala`
```

## 解答

TODO
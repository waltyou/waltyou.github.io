---
layout: post
title: 精通 Apache Spark 源码 | 问题 09・五、查询引擎 | Catalyst 查询优化（从分析到物理计划的全流程）
date: 2025-05-24 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第九个问题，属于第五大类"Catalyst与Tungsten引擎"，问题描述为：

```markdown
**Catalyst optimization phases**: Trace a query through analysis, logical optimization (e.g., predicate pushdown), physical planning, and code generation.  
   *Key files:* `RuleExecutor.scala`, `QueryExecution.scala`
```

## 解答

TODO
---
layout: post
title: 精通 Apache Spark 源码 | 问题 13・七、容错机制 | RDD 容错策略（血缘重算 vs 检查点实现）
date: 2025-05-28 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第十三个问题，属于第七大类"容错机制"，问题描述为：

```markdown
**Lineage vs checkpointing**: How does RDD recomputation work? When does lineage truncation occur in Structured Streaming?  
     *Key logic:* `RDD.checkpoint()`, `HDFSBackedStateStore`
```

## 解答

TODO
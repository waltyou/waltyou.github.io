---
layout: post
title: 精通 Apache Spark 源码 | 问题 11・六、内存管理 | 统一内存模型（执行/存储内存分配与溢写策略）
date: 2025-05-26 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第十一个问题，属于第六大类"内存管理"，问题描述为：

```markdown
**Unified memory model**: How does Spark balance execution vs storage memory? Trace spill-to-disk logic in ExternalSorter.  
     *Key classes:* `UnifiedMemoryManager`, `MemoryStore`
```

## 解答

TODO
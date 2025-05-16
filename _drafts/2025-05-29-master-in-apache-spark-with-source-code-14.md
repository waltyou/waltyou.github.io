---
layout: post
title: 精通 Apache Spark 源码 | 问题 14・七、容错机制 | Shuffle 故障恢复（MapOutputTracker 数据重建机制）
date: 2025-05-29 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第十四个问题，属于第七大类"容错机制"，问题描述为：

```markdown
**Shuffle fault recovery**: How do MapOutputTracker and BlockManager handle lost shuffle data during executor failures?
```

## 解答

TODO
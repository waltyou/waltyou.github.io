---
layout: post
title: 精通 Apache Spark 源码 | 问题 22・十一、高级优化 | Join 策略选择（BroadcastJoin 与 SortMergeJoin 阈值控制）
date: 2025-06-06 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第二十二个问题，属于第十一大类"高级优化"，问题描述为：

```markdown
**Join optimization**: Compare BroadcastJoin vs SortMergeJoin strategies. How is broadcast threshold determined?  
     *Key logic:* `JoinSelection.scala`, `BroadcastExchangeExec.scala`
```

## 解答

TODO
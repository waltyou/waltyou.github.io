---
layout: post
title: 精通 Apache Spark 源码 | 问题 07・四、Shuffle机制 | Shuffle 管理器对比（SortShuffle vs BypassMerge 的实现差异）
date: 2025-05-22 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第七个问题，属于第四大类"Shuffle与数据交换"，问题描述为：

```markdown
**Shuffle implementation**: Compare SortShuffleManager vs BypassMergeSortShuffleManager. How do ShuffleWriter/ShuffleReader handle data?  
   *Key files:* `ShuffleManager.scala`, `IndexShuffleBlockResolver.scala`
```

## 解答

TODO
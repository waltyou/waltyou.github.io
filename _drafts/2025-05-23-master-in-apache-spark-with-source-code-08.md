---
layout: post
title: 精通 Apache Spark 源码 | 问题 08・四、Shuffle机制 | Tungsten Shuffle 优化（堆外内存与缓存感知排序）
date: 2025-05-23 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第八个问题，属于第四大类"Shuffle与数据交换"，问题描述为：

```markdown
**Shuffle optimization**: How does Tungsten's UnsafeShuffleManager improve performance through off-heap memory and cache-aware sorting?
```

## 解答

TODO
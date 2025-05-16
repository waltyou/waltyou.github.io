---
layout: post
title: 精通 Apache Spark 源码 | 问题 12・六、内存管理 | Tungsten 内存优化（UnsafeRow 序列化与对象开销分析）
date: 2025-05-27 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第十二个问题，属于第六大类"内存管理"，问题描述为：

```markdown
**Tungsten memory format**: How does UnsafeRow optimize serialization/deserialization? Compare to Java object overhead.
```

## 解答

TODO
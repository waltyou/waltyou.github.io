---
layout: post
title: 精通 Apache Spark 源码 | 问题 04・二、RDD与抽象 | Dataset/DataFrame 内存优化（Encoders 与 Tungsten 格式转换）
date: 2025-05-19 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第四个问题，属于第二大类"RDD与数据抽象"，问题描述为：

```markdown
**Dataset/DataFrame internals**: How do Encoders bridge JVM objects to Tungsten's memory format?  
   *Key classes:* `ExpressionEncoder`, `RowEncoder`
```

## 解答

TODO
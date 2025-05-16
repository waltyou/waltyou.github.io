---
layout: post
title: 精通 Apache Spark 源码 | 问题 03・二、RDD与抽象 | RDD 血缘链实现（map/filter/join 转换的宽窄依赖对比）
date: 2025-05-18 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第三个问题，属于第二大类"RDD与数据抽象"，问题描述为：

```markdown
**RDD lineage implementation**: How do transformations (map/filter/join) create lineage chains? Compare narrow vs wide dependencies in code.  
   *Key files:* `RDD.scala`, `Dependency.scala`
```

## 解答

TODO
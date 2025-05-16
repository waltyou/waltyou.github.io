---
layout: post
title: 精通 Apache Spark 源码 | 问题 19・十、集群管理 | 动态资源分配（ExecutorAllocationManager 弹性扩缩容）
date: 2025-06-03 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第十九个问题，属于第十大类"集群管理"，问题描述为：

```markdown
**Dynamic allocation**: How does ExecutorAllocationManager scale resources based on task backlog?  
     *Key logic:* `ExecutorAllocationManager.scala`
```

## 解答

TODO
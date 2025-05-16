---
layout: post
title: 精通 Apache Spark 源码 | 问题 10・五、查询引擎 | 全阶段代码生成（WholeStageCodegenExec 原理与实践）
date: 2025-05-25 13:39:04
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

这是[本系列](../master-in-apache-spark-with-source-code-00)的第十个问题，属于第五大类"Catalyst与Tungsten引擎"，问题描述为：

```markdown
**Whole-stage codegen**: How does WholeStageCodegenExec collapse operator trees into single functions?  
     *Key files:* `WholeStageCodegenExec.scala`, `CodeGenerator.scala`
```

## 解答

TODO
---
layout: post
title: Apache Pig 入门学习
date: 2019-07-07 09:43:04
author: admin
comments: true
categories: [Pig]
tags: [Big Data, Pig]
---

工作中需要用到 Pig，来学习一下。

<!-- more -->

* 目录
{:toc}
---

# 1. 基本概念

## 是什么

官方文档定义：

> **Apache Pig** is a platform for analyzing large data sets that consists of a high-level language for expressing data analysis programs, coupled with infrastructure for evaluating these programs. The salient property of Pig programs is that their structure is amenable to substantial parallelization, which in turns enables them to handle very large data sets.

是一个使用高层次语言来分析大数据的平台。它显著的一个特点就是它的结构可以直接地进行并行计算，这个特点对处理大数据很有帮助。

它允许用户使用类-Sql 语言（Pig Latin）来进行数据处理，这个语言也提供了强大方便的 UDF 功能，所以它也可以集成其他语言的实现的功能。

它最终会将处理逻辑转换为 MapReduce Job进行运行。

## 出现的背景

1. 减小非开发人员处理大数据的学习成本。
2. 平时除了需要处理结构化数据（hive），非结构化数据也很常见。
3. 需要处理多种来源的数据



## 未完待续。。。
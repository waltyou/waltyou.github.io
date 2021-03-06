---
layout: post
title: 深入理解 Linux 内核（一）：内存寻址
date: 2020-04-28 18:11:04
author: admin
comments: true
categories: [Linux]
tags: [Linux, Linux Kernel]
---

进入书的第二章：内存寻址。

<!-- more -->

---

* 目录
{:toc}
---

## 内存地址
首先区分三类地址：
1. 逻辑地址：段（segment） + 偏移量（offset）
2. 线性地址：也称虚拟地址，是一个32位无符号整数
3. 物理地址：用于芯片级别内存单元寻址，通常与总线上的电信号相对应。

一次内存寻址，通常是以下过程： 

​	逻辑地址 -> [分段单元] -> 线性地址 -> [分页单元] -> 物理地址



## 硬件中的分段

### 1. 段选择符和段寄存器

段选择符就是逻辑地址中的第一部分，处理器提供段寄存器来存储段选择符。

### 2. 段描述符

它描述了段的特征，它放在全局描述符表或局部描述符中。

### 3. 分段单元

它将逻辑地址转换为线型地址。



## Linux 中的分段

分段可以给每个进程分配不同的线性地址空间，而分页可以把同一个线性地址空间映射到不同的物理地址空间。

Linux更喜欢分页方式，因为：

- 更加简单
- 使用分页更加通用



## 硬件中的分页

分页单元将线性地址转换为物理地址，其中一个关键任务是判断该请求是否有权限进行对应的访问类型，如果无效，产生一个缺页异常。

为了效率起见，线性地址被分为以固定长度为单位的组，成为页（page）。页内部连续的线性地址被影射到连续的物理地址上。

分页单元把所有RAM分成固定长度的页帧（物理页）。每个页帧包含一个页面，也就是页帧的大小和页面大小一样。页帧是主内存的组成部分，因此也是一块存储区域。区分页帧和页面很重要，页面是一块数据，它被存储在任意一个页帧上或者磁盘上。

把线性地址映射到物理地址的数据结构叫做页表；它存储在主存中，并被内核初始化（启用分页单元的情况下）。

## Linux中的分页

Linux采用一种可以兼容32位和64位系统的分页模型，

- 全局页目录
- 顶层页目录
- 中层页目录
- 页表

全局页目录包含若干顶层页目录的地址，顶层页目录又包含若干中层页目录的地址，中层页目录又包含一些页表的地址。每个页表节点指向一个页帧。因此，线性地址可以分成五个部分。


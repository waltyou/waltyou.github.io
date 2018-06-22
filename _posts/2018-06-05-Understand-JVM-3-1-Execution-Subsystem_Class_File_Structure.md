---
layout: post
title: 《深入理解Java虚拟机：JVM高级特性与最佳实践--第二版》学习日志（三）Part 1：类文件结构
date: 2018-06-05 20:06:04
author: admin
comments: true
categories: [Java]
tags: [Java，JVM]
---

书中的第三部分主要来了解一下虚拟机执行子系统。这部分有四个部分：类文件结构、虚拟机类加载机制、虚拟机字节码执行引擎和类加载及执行子系统的案例与实战。

我们先来看一下类文件结构。

<!-- more -->
---

学习资料主要参考： 《深入理解Java虚拟机：JVM高级特性与最佳实践(第二版)》，作者：周志明

---
## 目录
{:.no_toc}

* 目录
{:toc}

---

# 概述

Java刚刚诞生时，提出了一个非常著名的口号：“一次编写，到处运行（Write once， Run Anywhere）”。

这个口号的基础就是：不管是运行在什么平台上的虚拟机，都可以载入和执行同一种与平台无关的字节码。

当然这个字节码，不仅可以是Java语言的，也可以是其他语言。

---

# Class类文件的结构


Class文件是一组8位字节为基础单位的二进制流，各个数据项目严格按照顺序紧凑地排列在Class文件中，之间没有添加任何分隔符。

**基本数据类型**

存储结构中分为两种数据类型：无符号数和表。

1. 无符号数

    属于基本的数据类型，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个字节的无符号数。

    它可以用来描述数字、索引引用、数量值或者按照UTF-8编码构成字符串值。

2. 表

    由多个无符号数或者其他表作为数据项构成的复合数据类型。

    所有表都习惯地以“_info”结尾。

    表用于描述有层次关系的复合结构的数据。

## 1. 魔数与Class文件的版本

每个Class文件的头4个字节称为魔数。
它的唯一作用是确定这个文件是否为一个能被虚拟机接受的Class文件。
为：“0xCAFEBABE”

紧接着魔数的4个字节为Class文件的版本号：第5、6字节是次版本号，第7、8字节是主版本号。

## 2. 常量池

主次版本号后是常量池入口，常量池可以理解为Class文件之中的资源仓库，它是Class文件结构中与其他项目关联最多的数据类型，也是占用Class文件空间最大的的数据项目之一，同时它也是在Class文件中第一个出现的表类型数据项目。

常量池入口放置了一项u2类型的数据，来代表常量池容量计数值，这个计数值是从1开始的。所以常量数目是计数值减一。

常量池找那个存放两大类常量：字面量（Literal）和符号引用（Symbolic Reference）。字面量，如文本字符串、声明为final的常量值等。符号引用包括以上三类常量：
- 类和接口的全限定名
- 字段的名称和描述符
- 方法的名称和描述符

可以借助javap命令来帮助我们理解Class文件。

## 3. 访问标志

常量池之后，紧接着的两个字节代表访问标志。

这个标志用于识别一些类或者接口层次的访问信息。比如：这个Class是类还是接口？是否定义为public类型？是否为abstract类型？是否是final 类？

## 4. 类索引、父类索引与接口索引集合

类索引、父类索引都是一个u2类型的数据。接口索引集合是一组u2的数据结合。

Class文件通过这三项数据来确定这个类的继承关系。

## 5. 字段表集合

字段表用于描述接口或者类中声明的变量。字段包括类级变量以及实例级变量，但不包括在方法内部声明的局部变量。

包括的信息有：字段的作用域、实例变量还是类变量（static修饰符）、可变性（final）、并发可见性（volatile）、可否被序列化（transient修饰符）、字段数据类型、字段名称。字段类型和字段名称都是无法固定的，需要引用常量池中的常量来描述。其他的信息可以用标志位来表示。

## 6. 方法表集合

与字段表几乎一致，依次包括访问标志、名称索引、描述符索引、属性表集合。

## 7. 属性表集合

在Class文件、字段表、方法表都可以携带自己的属性表集合，以用于描述某些场景专有的信息。

---

# 字节码指令简介

Java虚拟机的指令由一个字节长度的、代表着某种特定操作含义的数字（操作码）以及跟随其后的零至多个代表此操作所需参数（操作数）而构成。

1. 字节码与数据类型
2. 加载和存储指令
3. 运算指令
4. 类型转换指令
5. 对象创建于访问指令
6. 操作数栈管理指令
7. 控制转移指令
8. 方法调用和返回指令
9. 异常处理指令
10. 同步指令
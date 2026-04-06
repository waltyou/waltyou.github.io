---
layout: post
title: 《Effective Java》学习日志（五）35：使用实例属性替代 ordinal
date: 2018-11-17 18:08:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

许多枚举通常与单个int值关联。所有枚举都有一个ordinal方法，它返回每个枚举常量类型的数值位置。

你可能想从序数中派生一个关联的int值：

    // Abuse of ordinal to derive an associated value - DON'T DO THIS
    public enum Ensemble {
        SOLO,   DUET,   TRIO, QUARTET, QUINTET,
        SEXTET, SEPTET, OCTET, NONET,  DECTET;
    
        public int numberOfMusicians() { return ordinal() + 1; }
    
    }
    
虽然这个枚举能正常工作，但对于维护来说则是一场噩梦。
如果常量被重新排序，numberOfMusicians方法将会中断。 
如果你想添加一个与你已经使用的int值相关的第二个枚举常量，则没有那么好运了。 

例如，为双四重奏（double quartet）添加一个常量可能会很好，它就像八重奏一样，由8位演奏家组成，但是没有办法做到这一点。

此外，如果没有给所有这些int值添加常量，也不能为某个int值添加一个常量。
例如，假设你想要添加一个常量，表示一个由12位演奏家组成的三重四重奏（triple quartet）。
对于由11个演奏家组成的合奏曲，并没有标准的术语，因此你不得不为未使用的int值（11）添加一个虚拟常量（dummy constant）。
最多看起来就是有些不好看。如果许多int值是未使用的，则是不切实际的。

幸运的是，这些问题有一个简单的解决方案。 
永远不要从枚举的序号中得出与它相关的值; 请将其保存在实例属性中：

    public enum Ensemble {
    
        SOLO(1), DUET(2), TRIO(3), QUARTET(4), QUINTET(5),
        SEXTET(6), SEPTET(7), OCTET(8), DOUBLE_QUARTET(8),
        NONET(9), DECTET(10), TRIPLE_QUARTET(12);
    
        private final int numberOfMusicians;
        Ensemble(int size) { this.numberOfMusicians = size; }
        public int numberOfMusicians() { return numberOfMusicians; }
    
    }
    
枚举规范对此ordinal方法说道：“大多数程序员对这种方法没有用处。 它被设计用于基于枚举的通用数据结构，如EnumSet和EnumMap。“

除非你在编写这样数据结构的代码，否则最好避免使用 ordinal 方法。
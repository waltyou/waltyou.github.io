---
layout: post
title: 《Effective Java》学习日志（五）36：使用 EnumSet 替代bit字段
date: 2018-11-19 18:10:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

如果枚举类型的元素主要用于集合中，一般来说使用int枚举模式.

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

下面将2的不同倍数赋值给每个常量：

    // Bit field enumeration constants - OBSOLETE!
    public class Text {
        public static final int STYLE_BOLD          = 1 << 0;  // 1
        public static final int STYLE_ITALIC        = 1 << 1;  // 2
        public static final int STYLE_UNDERLINE     = 1 << 2;  // 4
        public static final int STYLE_STRIKETHROUGH = 1 << 3;  // 8
    
        // Parameter is bitwise OR of zero or more STYLE_ constants
        public void applyStyles(int styles) { ... }
    }
    
这种表示方式允许你使用按位或（or）运算将几个常量合并到一个称为位属性（bit field）的集合中：

    text.applyStyles(STYLE_BOLD | STYLE_ITALIC);

位属性表示还允许你使用按位算术有效地执行集合运算，如并集和交集。 

但是位属性具有int枚举常量等的所有缺点。 

- 当打印为数字时，解释位属性比简单的int枚举常量更难理解。 没有简单的方法遍历所有由位属性表示的元素。 
- 必须预测在编写API时需要的最大位数，并相应地为位属性（通常为int或long）选择一种类型。 
一旦你选择了一个类型，你就不能超过它的宽度（32或64位）而不改变API。

一些程序员使用枚举优于int常量，当他们需要传递常量集合时仍然使用位属性。 
没有理由这样做，因为存在更好的选择。 

java.util包提供了 EnumSet 类来有效地表示从单个枚举类型中提取的值集合。 这个类实现了Set接口，提供了所有其他Set实现的丰富性，类型安全性和互操作性。 
但是在内部，每个EnumSet都表示为一个位矢量（bit vector）。 

如果底层的枚举类型有64个或更少的元素，并且大多数情况下，整个EnumSet用单个long表示，所以它的性能与位属性的性能相当。 
批量操作（如removeAll和retainAll）是使用按位算术实现的，就像你为位属性手动操作一样。 
但是完全避免了手动位混乱的丑陋和错误倾向：EnumSet为你做了很大的努力。

下面是前一个使用枚举和枚举集合替代位属性的示例。 它更短，更清晰，更安全：

    // EnumSet - a modern replacement for bit fields
    public class Text {
        public enum Style { BOLD, ITALIC, UNDERLINE, STRIKETHROUGH }
    
        // Any Set could be passed in, but EnumSet is clearly best
        public void applyStyles(Set<Style> styles) { ... }
    }
    
这里是将EnumSet实例传递给applyStyles方法的客户端代码。 
EnumSet类提供了一组丰富的静态工厂，可以轻松创建集合，其中一个代码如下所示：

    text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));

请注意，applyStyles方法采用Set<Style>而不是EnumSet<Style>参数。 
尽管所有客户端都可能会将EnumSet传递给该方法，但接受接口类型而不是实现类型通常是很好的做法（条目 64）。 
这允许一个不寻常的客户端通过其他Set实现的可能性。

总之，仅仅因为枚举类型将被用于集合中，所以没有理由用位属性来表示它。 
EnumSet类将位属性的简洁性和性能与条目 34中所述的枚举类型的所有优点相结合。
EnumSet的一个真正缺点是，它不像Java 9那样创建一个不可变的EnumSet，但是在即将发布的版本中可能会得到补救。 
同时，你可以用Collections.unmodifiableSet封装一个EnumSet，但是简洁性和性能会受到影响。

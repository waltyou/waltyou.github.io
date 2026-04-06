---
layout: post
title: 《Effective Java》学习日志（七）50：当需要时进行防御性复制
date: 2018-12-05 18:46:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

要小心类的使用者破坏类本生的安全性，所以适当的时候要进行防御性复制。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

让Java可以愉快使用的一件事是它是一种安全的语言。 
这意味着在没有native方法的情况下，它不受缓冲区溢出，数组溢出，野指针以及其他困扰C和C ++等不安全语言的内存损坏错误的影响。 
在一种安全的语言中，无论系统的任何其他部分发生什么，都可以编写类并确切地知道它们的不变量将保持不变。 
而这个特性，如果是在某种将所有内存视为一个巨型阵列的语言中，是不可能的。

即使是安全的语言，如果没有您的努力，您也不会与其的类隔离。 
**你必须假设这个类的用户会尽力摧毁它的不变量，来采取防御性的方案。** 
虽然有人会努力地破坏系统的安全性，但更常见的是，你的类将不得不应对善意程序员诚实错误导致的意外行为。 
无论哪种方式，值得花时间编写一些在不良行为客户面前表现强大的类。

虽然在没有对象帮助的情况下，另一个类不可能修改对象的内部状态，但是产生这种无意义的帮助却是令人惊讶的简单。 
例如，考虑以下类，它声称代表一个不可变的时间段：

    // Broken "immutable" time period class
    public final class Period {
        private final Date start;
        private final Date end;
        /**
        * @param start the beginning of the period
        * @param end the end of the period; must not precede start
        * @throws IllegalArgumentException if start is after end
        * @throws NullPointerException if start or end is null
        */
        public Period(Date start, Date end) {
            if (start.compareTo(end) > 0)
                throw new IllegalArgumentException(
                    start + " after " + end);
            this.start = start;
            this.end = end;
        }
        public Date start() {
            return start;
        }
        public Date end() {
            return end;
        }
        ...
        // Remainder omitted
    }

乍一看，这个类似乎是不可变的，并强制时间范围的开始不后于结束。 
但是，通过利用Date是可变的这一事实，很容易违反这个不变量：

    // Attack the internals of a Period instance
    Date start = new Date();
    Date end = new Date();
    Period p = new Period(start, end);
    end.setYear(78); // Modifies internals of p!

从Java 8开始，解决此问题的显而易见的方法是使用Instant（或Localor ZonedDateTime）代替Date，因为Instant（和其他java.time类）是不可变的（第17项）。 
**Date 已过时，不应再在新代码中使用。** 
也就是说，问题仍然存在：有时您必须在API和内部表示中使用可变值类型，并且此项中讨论的技术适用于这些时间。

为了保护Period实例的内部免受此类攻击，**必须将每个可变参数的防御性副本创建到构造函数**，并将这些副本用作Period实例的组件来代替原始文件：

    // Repaired constructor - makes defensive copies of parameters
    public Period(Date start, Date end) {
        this.start = new Date(start.getTime());
        this.end = new Date(end.getTime());
        if (this.start.compareTo(this.end) > 0)
            throw new IllegalArgumentException(
                this.start + " after " + this.end);
    }

使用新的构造函数，先前的攻击将对Period实例没有影响。
**请注意，在检查参数的有效性之前会制作防御性副本（第49项），并且对副本而不是原件执行有效性检查。**
虽然这可能看起来不自然，但这是必要的。
它可以在检查参数的时间和复制时间之间的漏洞窗口期间保护类不受另一个线程参数的更改。
在计算机安全社区，这被称为检查时间/使用时间或TOCTOU攻击[Viega01]。

另请注意，我们没有使用Date的克隆方法来制作防御性副本。
因为Date是非最终的，所以clone方法不能保证返回一个类为java.util.Date的对象：它可以返回一个专门为恶意恶作剧设计的不可信子类的实例。
例如，这样的子类可以在创建私有静态列表时记录对每个实例的引用，并允许攻击者访问该列表。
这将使攻击者可以自由控制所有实例。
**要防止此类攻击，请不要使用克隆方法来制作类型可由不信任方进行子类化的参数的防御副本。**

当替换构造函数成功抵御先前的攻击时，仍然可以改变Period实例，因为它的访问器提供对其可变内部的访问：

    // Second attack on the internals of a Period instance
    Date start = new Date();
    Date end = new Date();
    Period p = new Period(start, end);
    p.end().setYear(78); // Modifies internals of p!

为了抵御第二次攻击，**只需修改访问者以返回可变内部字段的防御性副本**：
    
    // Repaired accessors - make defensive copies of internal fields
    public Date start() {
        return new Date(start.getTime());
    }
    public Date end() {
        return new Date(end.getTime());
    }

使用新的构造函数和新的访问器，Period是真正不可变的。 
无论程序员多么恶意或无能，根本没有办法违反一个句号的开头不跟随其结束的不变量（不使用诸如本地方法和反射之类的语言学手段）。
这是正确的，因为除了Period本身之外的任何类都无法访问Period实例中的任何可变字段。 
这些字段真正封装在对象中。

在访问器中，与构造函数不同，允许使用克隆方法来制作防御性副本。 
这是因为我们知道Period的内部Date对象的类是java.util.Date，而不是一些不受信任的子类。 
也就是说，出于第13项中概述的原因，通常最好使用构造函数或静态工厂来复制实例。

防御性复制参数不仅适用于不可变类。
每次编写在内部数据结构中存储对客户端提供的对象的引用的方法或构造函数时，请考虑客户端提供的对象是否可能是可变的。
如果是，请考虑在将对象输入数据结构后，您的类是否可以容忍对象的更改。
如果答案为否，则必须防御性地复制对象并将副本输入数据结构中以代替原始对象。

例如，如果您正在考虑使用客户端提供的对象引用作为内部Set实例中的元素或作为内部Map实例中的键，您应该知道如果对象的集合或映射的不变量将被破坏插入后修改。

内部组件在返回客户端之前进行防御性复制也是如此。
无论您的类是否是不可变的，在返回对可变内部组件的引用之前，您应该三思而后行。
机会是，你应该返回一个防御性的副本。
请记住，非零长度数组总是可变的。
因此，在将内部数组返回给客户端之前，应始终制作防御性副本。或者，您可以返回数组的不可变视图。这两种技术都在第15项中示出。

可以说，所有这一切的真正教训是，在可能的情况下，您应该使用不可变对象作为对象的组件，这样您就不必担心防御性复制（第17项）。
在我们的Period示例中，使用Instant（或LocalDateTime或ZonedDateTime），除非您使用的是Java 8之前的版本。
如果您使用的是早期版本，则一个选项是存储Date.getTime返回的原始long代替Date引用。

可能存在与防御性复制相关的性能损失，并且它并不总是合理的。
如果一个类信任其调用者不修改内部组件，可能是因为该类及其客户端都是同一个包的一部分，那么放弃防御性复制可能是适当的。
在这些情况下，类文档应明确调用者不得修改受影响的参数或返回值。

即使跨越包边界，在将可变参数集成到对象之前制作可变参数的防御性副本并不总是合适的。
有一些方法和构造函数，其调用指示参数引用的对象的显式切换。
在调用这样的方法时，客户端承诺它将不再直接修改对象。
希望获得客户端提供的可变对象所有权的方法或构造函数必须在其文档中明确说明。

包含调用指示控制权转移的方法或构造函数的类无法抵御恶意客户端。
只有当一个类和它的客户之间存在相互信任，或者当对类的不变量造成损害时，除了客户之外，任何人都不会受到损害。
后一种情况的一个例子是包装类模式（第18项）。
根据包装类的性质，客户端可以通过在包装后直接访问对象来销毁类的不变量，但这通常只会损害客户端。

总之，如果一个类具有从其客户端获取或返回其客户端的可变组件，则该类必须防御性地复制这些组件。
如果副本的成本过高而且该类信任其客户不要不恰当地修改组件，则可以用文档替换防御性副本，该文档概述了客户不负责修改受影响组件的责任。


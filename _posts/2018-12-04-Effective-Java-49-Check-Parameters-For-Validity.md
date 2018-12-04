---
layout: post
title: 《Effective Java》学习日志（七）49：检查参数有效性
date: 2018-12-04 18:50:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

接下来进入了第7大章：方法。
这一章讨论了方法设计的几个方面：如何处理参数和返回值，如何设计方法签名以及如何记录方法。 
本章中的大部分内容适用于构造函数和方法。

先来看看如何处理参数。


<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---

## 目录
{:.no_toc}

* 目录
{:toc}

---

大多数方法和构造方法对可以将哪些值传递到其对应参数中有一些限制。 

例如，索引值必须是非负数，对象引用必须为非null。 
你应该清楚地在文档中记载所有这些限制，并在方法主体的开头用检查来强制执行。 
应该尝试在错误发生后尽快检测到错误，这是一般原则的特殊情况。 
如果不这样做，则不太可能检测到错误，并且一旦检测到错误就更难确定错误的来源。

如果将无效参数值传递给方法，并且该方法在执行之前检查其参数，则它抛出适当的异常然后快速且清楚地以失败结束。 

如果该方法无法检查其参数，可能会发生一些事情。 
在处理过程中，该方法可能会出现令人困惑的异常。 
更糟糕的是，该方法可以正常返回，但默默地计算错误的结果。 
最糟糕的是，该方法可以正常返回但是将某个对象置于受损状态，在将来某个未确定的时间在代码中的某些不相关点处导致错误。 
换句话说，验证参数失败可能导致违反故障原子性（failure atomicity ）（Item 76）。

对于公共方法和受保护方法，请使用Java文档@throws注解来记在在违反参数值限制时将引发的异常（Item 74）。 
通常，生成的异常是IllegalArgumentException，IndexOutOfBoundsException或NullPointerException（Item 72）。 
一旦记录了对方法参数的限制，并且记录了违反这些限制时将引发的异常，那么强制执行这些限制就很简单了。 

这是一个典型的例子：

    /**
    * Returns a BigInteger whose value is (this mod m). This method
    * differs from the remainder method in that it always returns a
    * non-negative BigInteger.
    *
    * @param m the modulus, which must be positive
    * @return this mod m
    * @throws ArithmeticException if m is less than or equal to 0
    */
    public BigInteger mod(BigInteger m) {
        if (m.signum() <= 0)
            throw new ArithmeticException("Modulus <= 0: " + m);
        ... // Do the computation
    }
    
请注意，文档注释没有说“如果m为null，mod抛出NullPointerException”，尽管该方法正是这样做的，这是调用m.sgn()的副产品。
这个异常记载在类级别文档注释中，用于包含的BigInteger类。
类级别的注释应用于类的所有公共方法中的所有参数。
这是避免在每个方法上分别记录每个NullPointerException的好方法。
它可以与@Nullable或类似的注释结合使用，以表明某个特定参数可能为空，但这种做法不是标准的，为此使用了多个注解。

在Java 7中添加的Objects.requireNonNull方法灵活方便，因此没有理由再手动执行空值检查。 
如果愿意，可以指定自定义异常详细消息。 
该方法返回其输入的值，因此可以在使用值的同时执行空检查：

    // Inline use of Java's null-checking facility
    this.strategy = Objects.requireNonNull(strategy, "strategy");

你也可以忽略返回值，并使用Objects.requireNonNull作为满足需求的独立空值检查。

在Java 9中，java.util.Objects类中添加了范围检查工具。 
此工具包含三个方法：checkFromIndexSize，checkFromToIndex和checkIndex。 
此工具不如空检查方法灵活。 
它不允许指定自己的异常详细消息，它仅用于列表和数组索引。 
它不处理闭合范围（包含两个端点）。 
但如果它能满足你的需要，那就很方便了。

对于未导出的方法，作为包的作者，控制调用方法的环境，这样就可以并且应该确保只传入有效的参数值。因此，非公共方法可以使用断言检查其参数，如下所示:

    // Private helper function for a recursive sort
    private static void sort(long a[], int offset, int length) {
        assert a != null;
        assert offset >= 0 && offset <= a.length;
        assert length >= 0 && length <= a.length - offset;
        ... // Do the computation
    }

本质上，这些断言声称断言条件将成立，无论其客户端如何使用封闭包。
与普通的有效性检查不同，断言如果失败会抛出AssertionError。
与普通的有效性检查不同的是，除非使用-ea(或者-enableassertions）标记传递给java命令来启用它们，否则它们不会产生任何效果，本质上也不会产生任何成本。
有关断言的更多信息，请参阅教程assert。

检查方法中未使用但存储以供以后使用的参数的有效性尤为重要。
例如，考虑第101页上的静态工厂方法，它接受一个int数组并返回数组的List视图。
如果客户端传入null，该方法将抛出NullPointerException，因为该方法具有显式检查(调用Objects.requireNonNull方法)。
如果省略了该检查，则该方法将返回对新创建的List实例的引用，该实例将在客户端尝试使用它时立即抛出NullPointerException。 
到那时，List实例的来源可能很难确定，这可能会使调试任务大大复杂化。

构造方法是这个原则的一个特例，你应该检查要存储起来供以后使用的参数的有效性。
检查构造方法参数的有效性对于防止构造对象违反类不变性（class invariants）非常重要。

你应该在执行计算之前显式检查方法的参数，但这一规则也有例外。 
一个重要的例外是有效性检查昂贵或不切实际的情况，并且在进行计算的过程中隐式执行检查。 

例如，考虑一种对对象列表进行排序的方法，例如Collections.sort(List)。 列表中的所有对象必须是可相互比较的。 
在对列表进行排序的过程中，列表中的每个对象都将与其他对象进行比较。 
如果对象不可相互比较，则某些比较操作抛出ClassCastException异常，这正是sort方法应该执行的操作。 
因此，提前检查列表中的元素是否具有可比性是没有意义的。 
但请注意，不加选择地依赖隐式有效性检查会导致失败原子性（ failure atomicity）的丢失（Item 76）。

有时，计算会隐式执行必需的有效性检查，但如果检查失败则会抛出错误的异常。 
换句话说，计算由于无效参数值而自然抛出的异常与文档记录方法抛出的异常不匹配。 
在这些情况下，你应该使用Item 73中描述的异常翻译（ exception translation）习惯用法将自然异常转换为正确的异常。

不要从本Item中推断出对参数的任意限制都是一件好事。 
相反，你应该设计一些方法，使其尽可能通用。 
假设方法可以对它接受的所有参数值做一些合理的操作，那么对参数的限制越少越好。 
但是，通常情况下，某些限制是正在实现的抽象所固有的。

总而言之，每次编写方法或构造方法时，都应该考虑对其参数存在哪些限制。 
应该记在这些限制，并在方法体的开头使用显式检查来强制执行这些限制。 
养成这样做的习惯很重要。 
在第一次有效性检查失败时，它所需要的少量工作将会得到对应的回报。
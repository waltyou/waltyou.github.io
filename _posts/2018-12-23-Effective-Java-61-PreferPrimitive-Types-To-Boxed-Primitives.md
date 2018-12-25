---
layout: post
title: 《Effective Java》学习日志（八）61:使用原始类型而不是盒装基元
date: 2018-12-23 14:22:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---



<!-- more -->

------

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

------

## 目录
{:.no_toc}

* 目录
{:toc}
------

Java有一个由两部分组成的类型系统，由基元组成，如int，double和boolean，以及引用类型，如String和List。每个基本类型都有一个相应的引用类型，称为盒装基元。对应于int，double和boolean的盒装基元是Integer，Double和Boolean。

如第6项所述，自动装箱和自动拆箱模糊，但不删除基元和盒装基元类型之间的区别。两者之间存在着真正的差异，重要的是要始终了解自己正在使用哪种以及在它们之间谨慎选择。

基元和盒装基元之间存在三个主要差异。

首先，基元只有它们的值，而盒装基元的标识与它们的值不同。换句话说，两个盒装原始实例可以具有相同的值和不同的身份。其次，原始类型只有完全功能的值，而每个盒装基元类型除了相应原始类型的所有功能值外，还有一个非功能值，即null。最后，基元比盒装基元更具时间和空间效率。如果你不小心，这三种差异都会让你陷入困境。

考虑以下比较器，该比较器旨在表示整数值的升序数字顺序。 （回想一下，比较器的compare方法返回一个负数，零或正数，具体取决于它的第一个参数是小于，等于还是大于它的第二个参数。）你不需要在实践中编写这个比较器因为它实现了Integer的自然排序，但它有一个有趣的例子：

```java
// Broken comparator - can you spot the flaw?
Comparator<Integer> naturalOrder =
		(i, j) -> (i < j) ? -1 : (i == j ? 0 : 1);
```

这个比较器看起来应该可以工作，它将通过许多测试。例如，它可以与Collections.sort一起使用，以正确排序百万元素列表，无论列表是否包含重复元素。但比较器存在严重缺陷。为了展示这个缺陷，只需打印naturalOrder.compare的值（new Integer（42），new Integer（42））。两个Integer实例都表示相同的值（42），因此该表达式的值应为0，但它为1，表示第一个Integer值大于第二个值！

所以有什么问题？ naturalOrder中的第一个测试工作正常。评估表达式i <j会导致i和j引用的Integer实例自动取消装箱;也就是说，它提取原始值。评估继续进行以检查结果int值中的第一个是否小于第二个。但是假设它不是。然后，下一个测试将计算表达式i == j，它对两个对象引用执行标识比较。如果i和j引用表示相同int值的不同Integer实例，则此比较将返回false，并且比较器将错误地返回1，表示第一个Integer值大于第二个值。**将==运算符应用于盒装基元几乎总是错误的。**

在实践中，如果你需要一个比较器来描述一个类型的自然顺序，你应该简单地调用Comparator.naturalOrder（），如果你自己编写一个比较器，你应该使用比较器构造方法，或原始的静态比较方法类型（第14项）。也就是说，您可以通过添加两个局部变量来存储与盒装的Integer参数对应的原始int值，并对这些变量执行所有比较，从而解决损坏的比较器中的问题。这避免了错误的身份比较：

```java
Comparator<Integer> naturalOrder = (iBoxed, jBoxed) -> {
    int i = iBoxed, j = jBoxed; // Auto-unboxing
    return i < j ? -1 : (i == j ? 0 : 1);
};
```

接下来，考虑这个令人愉快的小程序：

```java
public class Unbelievable {
    static Integer i;
    
    public static void main(String[] args) {
        if (i == 42)
            System.out.println("Unbelievable");
    }
}
```

不，它不打印“Unbelievable” - 但它的作用几乎同样奇怪。 在计算表达式i == 42时，它会抛出NullPointerException。 问题是我是一个Integer，而不是一个int，和所有非常量对象引用字段一样，它的初始值为null。 当程序计算表达式i == 42时，它将Integer与int进行比较。 几乎在每种情况下，**当您在操作中混合基元和盒装基元时，盒装基元将自动取消装箱**。 如果空对象引用是自动取消装箱的，则会出现NullPointerException。 正如该计划所示，它几乎可以在任何地方发生。 解决问题就像声明我是一个int而不是一个Integer一样简单。

最后，请考虑第6项中第24页的程序：

```java
// Hideously slow program! Can you spot the object creation?
public static void main(String[] args) {
    Long sum = 0L;
    for (long i = 0; i < Integer.MAX_VALUE; i++) {
    	sum += i;
    }
    System.out.println(sum);
}
```

这个程序比它应该慢得多，因为它意外地声明一个局部变量（sum）是盒装基元类型Long而不是基本类型long。程序编译时没有错误或警告，并且变量被重复加框和取消装箱，导致观察到的性能下降。

在本项目讨论的所有三个程序中，问题都是相同的：程序员忽略了原语和盒装原语之间的区别，并遭受了后果。在前两个计划中，后果是完全失败;在第三，严重的性能问题。

那么什么时候应该使用盒装原语？它们有几种合法用途。第一个是集合中的元素，键和值。您不能将基元放在集合中，因此您不得不使用盒装基元。这是一个更普遍的特例。您必须在参数化类型和方法（第5章）中使用盒装基元作为类型参数，因为该语言不允许您使用基元。例如，您不能将变量声明为ThreadLocal <int>类型，因此您必须使用ThreadLocal <Integer>。最后，在进行反射方法调用时必须使用盒装基元（第65项）。

总之，只要有选择，就可以优先使用基元而不是盒装基元。原始类型更简单，更快捷。如果你必须使用盒装基元，小心！**自动装箱减少了使用盒装基元的冗长度，但没有降低危险性**。当你的程序将两个盒装基元与==运算符进行比较时，它会进行身份比较，这几乎肯定不是你想要的。当您的程序执行涉及盒装和未装箱原语的混合类型计算时，它会进行拆箱，**当您的程序进行拆箱时，它会抛出NullPointerException**。最后，当您的程序框原始值时，它可能导致代价高昂且不必要的对象创建。
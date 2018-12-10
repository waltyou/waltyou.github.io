---
layout: post
title: 《Effective Java》学习日志（七）53：谨慎地使用变长参数
date: 2018-12-10 15:43:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

当需要使用可变数量的参数定义方法时，varargs非常有用。 

只需要注意两点：在varargs参数前加上任何必需的参数；注意使用varargs的性能后果。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---

## 目录
{:.no_toc}

* 目录
{:toc}
---

Varargs方法，正式称为变量arity方法[JLS，8.4.1]，接受零个或多个指定类型的参数。 varargs工具首先创建一个数组，其大小是在调用站点传递的参数数量，然后将参数值放入数组中，最后将数组传递给方法。

例如，这是一个varargs方法，它接受一系列int参数并返回它们的总和。 正如您所料，sum（1,2,3）的值为6，sum() 的值为0：

```java
// Simple use of varargs
static int sum(int... args) {
    int sum = 0;
    for (int arg : args)
    	sum += arg;
    return sum;
}
```

有时候编写一个需要一个或多个相同类型参数的方法是合适的，但是参数数目如果是0就不合适了。 

例如，假设您要编写一个计算其参数最小值的函数。 如果客户端不传递任何参数，则此函数定义不明确。 您可以在运行时检查数组长度：

```java
// The WRONG way to use varargs to pass one or more arguments!
static int min(int... args) {
    if (args.length == 0)
    	throw new IllegalArgumentException("Too few arguments");
    int min = args[0];
    for (int i = 1; i < args.length; i++)
        if (args[i] < min)
        	min = args[i];
    return min;
}
```

该解决方案存在几个问题。 

最严重的是，如果客户端在没有参数的情况下调用此方法，则它会在运行时而不是编译时失败。

另一个问题是它很难看。 您必须在args上包含显式有效性检查，除非将min初始化为Integer.MAX_VALUE，否则不能使用for-each循环，但这也很难看。

幸运的是，有一种更好的方法可以达到预期的效果。 

声明方法采用两个参数，一个指定类型的普通参数和一个此类型的varargs参数。 该解决方案纠正了前一个方面的所有缺陷：

```java
// The right way to use varargs to pass one or more arguments
static int min(int firstArg, int... remainingArgs) {
    int min = firstArg;
    for (int arg : remainingArgs)
        if (arg < min)
        	min = arg;
    return min;
}
```

从这个例子中可以看出，varargs在你想要一个带有可变数量参数的方法的情况下是有效的。 Varargs专为printf设计，与varargs同时添加到平台，以及改装的核心反射设施（项目65）。

 printf和反射都从varargs中获益匪浅。

在性能危急情况下使用varargs时要小心。 每次调用varargs方法都会导致数组分配和初始化。 如果你根据经验确定你负担不起这个费用但是你需要varargs的灵活性，那么你可以通过一个模式匹配来完成你的所想。

假设您已确定95％的方法调用具有三个或更少的参数。 然后声明方法的五个重载，一个用零到三个普通参数，以及一个varargs方法，当参数个数超过三个时使用：

```java
public void foo() { }
public void foo(int a1) { }
public void foo(int a1, int a2) { }
public void foo(int a1, int a2, int a3) { }
public void foo(int a1, int a2, int a3, int... rest) { }
```

现在您知道，只有在参数数量超过3的所有调用的5％中，您才需要支付数组创建的成本。 像大多数性能优化一样，这种技术通常并不合适，但是当它合适的时候，它就是我们的救星。

EnumSet的静态工厂使用此技术将创建枚举集的成本降至最低。 这是合适的，因为枚举集替换bit字段后，提供了性能优势（第36项）。

总之，当您需要使用可变数量的参数定义方法时，varargs非常有用。 在varargs参数前加上任何必需的参数，并注意使用varargs的性能后果。

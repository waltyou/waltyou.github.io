---
layout: post
title: 《Effective Java》学习日志（九）75:在详细消息中包含failure-capture信息
date: 2019-01-13 10:06:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]


---



<!-- more -->

------

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

------




* 目录
{:toc}

------

当程序因未捕获的异常而失败时，系统会自动打印出异常的堆栈跟踪。 堆栈跟踪包含异常的字符串表示形式，即调用其toString方法的结果。 这通常包含异常的类名，后跟其详细消息。 通常，这是程序员或站点可靠性工程师在调查软件故障时将获得的唯一信息。 如果故障不易再现，则可能很难或不可能获得更多信息。 因此，异常的toString方法返回尽可能多的关于失败原因的信息是至关重要的。 换句话说，异常的详细消息应该捕获后续分析的失败。

**要捕获失败，异常的详细消息应包含导致异常的所有参数和字段的值**。 例如，IndexOutOfBoundsException的详细消息应包含下限，上限和未能位于边界之间的索引值。 这些信息告诉了很多关于失败的信息。 三个值中的任何一个或全部都可能是错误的。 索引可能比下限小一个或等于上限（“fencepost error”），或者它可能是一个狂野值，太低或太高。 下限可能大于上限（严重的内部不变失败）。 这些情况中的每一种都指向一个不同的问题，如果您知道您正在寻找什么样的错误，它将极大地帮助您进行诊断。

一个警告涉及安全敏感信息。由于许多人在诊断和修复软件问题的过程中可能会看到堆栈跟踪，**因此请不要在详细消息中包含密码，加密密钥等**。

虽然将所有相关数据包含在例外的详细信息中至关重要，但通常包含大量散文并不重要。堆栈跟踪旨在与文档一起进行分析，并在必要时进行源代码分析。它通常包含引发异常的确切文件和行号，以及堆栈上所有其他方法调用的文件和行号。冗长的散文描述失败是多余的;通过阅读文档和源代码可以收集信息。

不应将异常的详细消息与用户级错误消息混淆，后者必须能够为最终用户理解。与用户级错误消息不同，详细消息主要是为了程序员或站点可靠性工程师在分析故障时的利益。因此，信息内容远比可读性重要。用户级错误消息通常是本地化的，而异常详细消息很少。

确保异常在其详细消息中包含足够的故障捕获信息的一种方法是在其构造函数中使用此信息而不是字符串详细消息。 然后可以自动生成详细消息以包括信息。 例如，IndexOutOfBoundsException可能有一个如下所示的构造函数，而不是String构造函数：

```java
/**
* Constructs an IndexOutOfBoundsException.
*
* @param lowerBound the lowest legal index value
* @param upperBound the highest legal index value plus one
* @param index
the actual index value
*/
public IndexOutOfBoundsException(int lowerBound, int upperBound,
    							int index) {
    // Generate a detail message that captures the failure
    super(String.format(
            "Lower bound: %d, Upper bound: %d, Index: %d",
            lowerBound, upperBound, index));
    // Save failure information for programmatic access
    this.lowerBound = lowerBound;
    this.upperBound = upperBound;
    this.index = index;
}
```

从Java 9开始，IndexOutOfBoundsException最终获得了一个带有int值索引参数的构造函数，但遗憾的是它省略了lowerBound和upperBound参数。 更一般地说，Java库并没有大量使用这个习惯用法，但强烈建议使用它。 它使程序员可以轻松抛出异常来捕获故障。 事实上，它使程序员难以捕获失败！ 实际上，成语集中了代码以在异常类中生成高质量的详细消息，而不是要求类的每个用户冗余地生成详细消息。

如第70项所示，异常可能适合为其失败捕获信息（上例中的lowerBound，upperBound和index）提供访问器方法。 在已检查的异常上提供此类访问器方法比未选中更为重要，因为故障捕获信息可用于从故障中恢复。 程序员可能希望以编程方式访问未经检查的异常的细节，这是罕见的（尽管不是不可思议）。 然而，即使对于未经检查的例外情况，似乎建议根据一般原则提供这些访问者（第12项，第57页）。


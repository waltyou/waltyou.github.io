---
layout: post
title: 《Effective Java》学习日志（九）77:不要忽略异常
date: 2019-01-13 10:30:04
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

虽然这个建议看起来很明显，但它经常受到侵犯，以至于它重复出现。 当API的设计者声明一个抛出异常的方法时，他们会试图告诉你一些事情。 不要忽视它！ 通过使用try块为空的try语句围绕方法调用，可以很容易地忽略异常：

```java
// Empty catch block ignores exception - Highly suspect!
try {
	...
} catch (SomeException e) {
}
```

**空的捕获块会使异常的目的失效，这会迫使您处理异常情况**。忽略异常类似于忽略火警 - 并将其关闭，这样就没有其他人有机会看到是否有真火。你可能会逃避它，或者结果可能是灾难性的。每当你看到一个空的挡块时，你的头上就会响起警铃。

在某些情况下，忽略异常是合适的。例如，关闭FileInputStream可能是合适的。您尚未更改文件的状态，因此无需执行任何恢复操作，并且您已经从文件中读取了所需的信息，因此没有理由中止正在进行的操作。记录异常可能是明智的，这样如果经常发生这些异常，您就可以调查此事。**如果您选择忽略异常，catch块应该包含一个注释，解释为什么这样做是合适的，并且该变量应该被命名为ignored**：

```java
Future<Integer> f = exec.submit(planarMap::chromaticNumber);
int numColors = 4; // Default; guaranteed sufficient for any map
try {
	numColors = f.get(1L, TimeUnit.SECONDS);
} catch (TimeoutException | ExecutionException ignored) {
	// Use default: minimal coloring is desirable, not required
}
```

此项目中的建议同样适用于已检查和未检查的异常。 无论异常是代表可预测的异常情况还是编程错误，使用空catch块忽略它都会导致程序在出现错误时以静默方式继续运行。 然后，程序可能在将来的任意时间失败，代码中的某一点与问题的根源没有明显的关系。 正确处理异常可以完全避免失败。 仅仅让异常向外传播至少会导致程序迅速失败，保留信息以帮助调试失败。
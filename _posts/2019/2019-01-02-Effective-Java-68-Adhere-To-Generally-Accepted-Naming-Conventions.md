---
layout: post
title: 《Effective Java》学习日志（八）68:遵守普遍接受的命名惯例
date: 2019-01-02 18:42:04
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

Java平台有一套完善的命名约定，其中许多都包含在Java语言规范[JLS，6.1]中。 简而言之，命名约定分为两类：印刷（typographical）和语法（grammatical）。

只有少数印刷命名约定，包括包，类，接口，方法，字段和类型变量。 你应该很少违反它们，永远不会没有充分的理由。 如果API违反这些约定，则可能难以使用。 如果实施违反了它们，则可能难以维护。 在这两种情况下，违规都有可能混淆和激怒使用代码的其他程序员，并可能导致错误的错误假设。 这些公约在本项目中进行了总结。

包和模块名称应该是分层的，组件由句点分隔。 组件应包含小写字母字符，很少包含数字。 将在您的组织外部使用的任何程序包的名称应以您组织的Internet域名开头，其组件相反，例如，edu.cmu，com.google，org.eff。 名称以java和javax开头的标准库和可选包是此规则的例外。 用户不得创建名称以java或javax开头的包或模块。 可以在JLS [JLS，6.1]中找到将Internet域名转换为包名称前缀的详细规则。

包名称的其余部分应包含一个或多个描述包的组件。 组件应该很短，通常是八个或更少的字符。 鼓励使用有意义的缩写，例如，util而不是utilities 。 缩略语是可以接受的，例如，awt。 组件通常应由单个单词或缩写组成。

除了Internet域名之外，许多包的名称只包含一个组件。 其他组件适用于大型设施，这些设施的大小要求将它们分解为非正式的层次结构。 例如，javax.util包具有丰富的包层次结构，其名称如java.util.concurrent.atomic。 这样的包被称为子包，尽管对包层次结构几乎没有语言支持。

类和接口名称（包括枚举和注释类型名称）应由一个或多个单词组成，每个单词的首字母大写，例如List或FutureTask。 除了首字母缩略词和某些常用缩写（如max和min）之外，应避免使用缩写。 关于首字母缩略词是大写还是只有首字母大写，存在一些分歧。 虽然一些程序员仍然使用大写字母，但是可以强有力地论证只有第一个字母大写：即使多个首字母缩略词背靠背出现，你仍然可以知道一个单词的起始位置和下一个单词的结尾。 您更喜欢看哪个类名，HTTPURL或HttpUrl？

方法和字段名称遵循与类和接口名称相同的排版约定，但方法或字段名称的第一个字母应为小写，例如remove或ensureCapacity。 如果首字母缩略词作为方法或字段名称的第一个单词出现，则它应该是小写的。

以前规则的唯一例外是“常量字段”，其名称应由一个或多个由下划线字符分隔的大写单词组成，例如VALUES或NEGATIVE_INFINITY。 常量字段是静态最终字段，其值是不可变的。 如果静态final字段具有基本类型或不可变引用类型（第17项），则它是常量字段。 例如，枚举常量是常量字段。 如果静态final字段具有可变引用类型，则如果引用的对象是不可变的，则它仍然可以是常量字段。 请注意，常量字段构成了下划线的唯一推荐用法。

局部变量名称与成员名称具有相似的排版命名约定，但允许使用缩写除外，单个字符和短字符序列的含义取决于它们出现的上下文，例如i，denom，houseNum。 输入参数是一种特殊的局部变量。 它们的名称应该比普通的局部变量更加仔细，因为它们的名称是其方法文档中不可或缺的一部分。

类型参数名称通常由单个字母组成。 最常见的是它是以下五种中的一种：T代表任意类型，E代表集合的元素类型，K代表V和V代表地图的值类型，X代表异常。 函数的返回类型通常是R. 任意类型的序列可以是T，U，V或T1，T2，T3。

为便于快速参考，下表显示了印刷约定的示例。

| 类型               | 例子                                              |
| ------------------ | ------------------------------------------------- |
| Package or module  | org.junit.jupiter.api , com.google.common.collect |
| Class or Interface | Stream , FutureTask , LinkedHashMap , HttpClient  |
| Method or Field    | remove , groupingBy , getCrc                      |
| Constant Field     | MIN_VALUE , NEGATIVE_INFINITY                     |
| Local Variable     | i , denom , houseNum                              |
| Type Parameter     | T , E , K , V , X , R , U , V, T1, T2             |

语法命名约定比印刷约定更灵活，更具争议性。 对于包而言，没有语法命名约定。 可实例化的类（包括枚举类型）通常以单数名词或名词短语命名，例如Thread，PriorityQueue或ChessPiece。 不可实例化的实用程序类（第4项）通常以复数名词命名，例如收集器或集合。 接口被命名为类，例如，Collection或Comparator，或者以能够或者为结尾的形容词，例如，Runnable，Iterable或Accessible。 由于anno- tation类型有如此多的用途，因此没有任何词性占主导地位。 名词，动词，介词和形容词都很常见，例如BindingAnnotation，Inject，ImplementedBy或Singleton。

返回非布尔函数或调用它们的对象的属性的方法通常以名词，名词短语或以动词get开头的动词短语命名，例如size，hashCode或getTime。 有一个声乐队伍声称只有第三种形式（以get开头）是可以接受的，但这种说法几乎没有基础。 前两种形式通常会产生更易读的代码，例如：

```java
if (car.speed() > 2 * SPEED_LIMIT)
	generateAudibleAlert("Watch out for cops!");
```

以get开头的表单源于大部分过时的Java Beans规范，该规范构成了早期可重用组件体系结构的基础。 有一些现代工具继续依赖于Beans命名约定，您可以随意在任何与这些工具结合使用的代码中使用它。 如果类包含同一属性的setter和getter，则遵循此命名约定的先例也很强。 在这种情况下，这两种方法通常命名为get Attribute并设置Attribute。

一些方法名称值得特别提及。转换对象类型，返回不同类型的独立对象的实例方法通常调用Type，例如toString或toArray。返回类型与接收对象类型不同的视图（第6项）的方法通常称为Type，GENERAL PROGRAMMING，例如asList。返回与调用它们的对象具有相同值的原语的方法通常称为类型Value，例如intValue。静态工厂的通用名称包括from，of，valueOf，instance，getInstance，newInstance，get Type和new Type（Item 1，page 9）。

字段名称的语法约定不太完善，并且不如类，接口和方法名称那么重要，因为精心设计的API包含很少（如果有）暴露字段。 boolean类型的字段通常被命名为boolean accessor方法，省略了initial，例如，initialized，composite。其他类型的字段通常以名词或名词短语命名，例如height，digits或bodyStyle。局部变量的语法约定类似于字段，但甚至更弱。

总而言之，内化标准命名约定并学习使用它们作为第二天性。印刷惯例很简单，很明确;语法惯例更复杂，更宽松。引用Java语言规范[JLS，6.1]，“如果长期使用常规用法指示其他方面，则不应盲目遵循这些约定。”使用常识。


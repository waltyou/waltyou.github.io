---
layout: post
title: 《Effective Java》学习日志（六）44：优先使用标准函数接口
date: 2018-11-28 18:40:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

大多数情况下，我们无需定义自己的函数接口，java.util.function包提供了大量标准函数接口供我们使用。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

既然Java有lambda，那么编写API的最佳实践已经发生了很大变化。

例如，模板方法模式[Gamma95]，其中子类重写基本方法以专门化其超类的行为，远没有那么吸引人。
现代的替代方法是提供一个静态工厂或构造函数，它接受一个函数对象来实现相同的效果。
更一般地说，你会编写更多以函数对象作为参数的构造函数和方法。选择正确的函数参数类型需要谨慎。

考虑LinkedHashMap。
您可以通过覆盖其受保护的removeEldestEntry方法将此类用作缓存，该方法每次将新键添加到 map 时都会调用。
当此方法返回true时，映射将删除其最旧的条目，该条目将传递给该方法。
以下覆盖允许映射增长到一百个条目，然后在每次添加新密钥时删除最旧的条目，从而保留最近的一百个条目：

    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return size() > 100;
    }

这种技术很好，但你可以用lambdas做得更好。 
如果今天写了LinkedHashMap，它将有一个带有函数对象的静态工厂或构造函数。 
查看removeEldestEntry的声明，您可能会认为函数对象应该使用Map.Entry <K，V>并返回一个布尔值，但这样做不会：
removeEldestEntry方法调用size（）来获取 map 中的条目的数目，因为removeEldestEntry是map上的实例方法。 
传递给构造函数的函数对象不是 map 上的实例方法，并且无法捕获它，因为在调用其工厂或构造函数时映射尚不存在。 
因此，映射必须将自身传递给函数对象，因此函数对象必须在输入及其最旧条目上获取映射。 
如果你要声明这样一个函数接口，它看起来像这样：

    // Unnecessary functional interface; use a standard one instead.
    @FunctionalInterface interface EldestEntryRemovalFunction<K,V>{
        boolean remove(Map<K,V> map, Map.Entry<K,V> eldest);
    }

此接口可以正常工作，但您不应该使用它，因为您不需要为此目的声明新接口。 
java.util.function包提供了大量标准函数接口供您使用。

**如果其中一个标准函数接口完成了这项工作，您通常应该优先使用它，而不是专门构建的函数接口。**

这将使您的API更容易学习，通过减少其概念表面积，并将提供显着的互操作性优势，因为许多标准函数接口提供有用的默认方法。 
例如，Predicate接口提供了组合谓词的方法。 
对于LinkedHashMap示例，应优先使用标准 BiPredicate<Map<K,V>, Map.Entry<K,V>> 接口，而不是自定义EldestEntryRemovalFunction接口。

java.util.Function中有四十三个接口。 
不能指望你记住它们，但是如果你记得六个基本接口，你可以在需要时得到它们。 
基本接口对对象引用类型进行操作。 
- Operator接口表示结果和参数类型相同的函数。 
- Predicate接口表示一个接受参数并返回布尔值的函数。 
- Function接口表示其参数和返回类型不同的函数。
- Supplier接口表示不带参数并返回（或“提供”）值的函数。 
- Consumer表示一个函数，它接受一个参数并且什么都不返回，基本上消耗它的参数。 

六个基本函数接口总结如下：

[![](/images/posts/effective-java-44.jpg)](/images/posts/effective-java-44.jpg)

六种基本接口中的每一种都有三种变体可以对基本类型int，long和double进行操作。 
它们的名称来源于基本接口，前缀为基本类型。 
因此，例如，带有int的谓词是IntPredicate，带有两个long值并返回long的二元运算符是LongBinaryOperator。 
除函数变量外，这些变量类型都不参数化，函数变量由返回类型参数化。 
例如，LongFunction<int[]> 取一个long并返回一个 int[]。

Function接口有九个附加变体，供结果类型为原始时使用。
源和结果类型总是不同，因为从类型到自身的函数是UnaryOperator。

如果源类型和结果类型都是原始类型，则使用Src To Result作为前缀Function，例如LongToIntFunction（六个变体）。
如果源是基元并且结果是对象引用，则使用<Src> ToObj作为前缀Function，例如DoubleToObjFunction（三个变体）。
三个基本函数接口有两个参数版本，它们是有意义的：BiPredicate <T，U>，BiFunction <T，U，R>和BiConsumer <T，U>。
还有BiFunction变体返回三个相关的基本类型：ToIntBiFunction <T，U>，ToLongBiFunction <T，U>和ToDoubleBiFunction <T，U>。 
Consumer的两个参数变体采用一个对象引用和一个基本类型：ObjDoubleConsumer <T>，ObjIntConsumer <T>和ObjLongConsumer <T>。
总共有九个基本接口的参数版本。

最后，还有BooleanSupplier接口，这是Supplier的一个变量，它返回布尔值。
这是任何标准函数接口名称中唯一明确提到的布尔类型，但是通过Predicate及其四种变体形式支持布尔返回值。 
BooleanSupplier接口和前面段落中描述的四十二个接口占所有四十三个标准函数接口。
不可否认，这是一个很大的吞并，而不是非常正交。
另一方面，您需要的大部分函数接口都是为您编写的，并且它们的名称足够常规，以便您在需要时不会遇到太多麻烦。

大多数标准函数接口仅用于提供对原始类型的支持。
不要试图使用有装箱类型的基本函数接口替代原始函数接口。
虽然它有效但它违反了第61条的建议，“更喜欢原始类型而不是装箱类型。”
使用装箱类型进行批量操作的性能后果可能是致命的。
现在您知道通常应该使用标准函数接口而不是编写自己的接口。但什么时候应该自己写？
当然，如果没有标准的那些符合您的需要，您需要自己编写，例如，如果您需要一个带有三个参数的谓词，或者一个抛出已检查异常的谓词。
但有时你应该编写自己的函数接口，即使其中一个标准结构完全相同。

考虑我们的老朋友Comparator <T>，它在结构上与ToIntBiFunction <T，T>接口相同。
即使后者接口已经存在，当前者被添加到库中时，使用它也是错误的。 
Comparator有几个原因值得拥有自己的接口。
1. 它的名称每次在API中使用时都提供了出色的文档，而且它的使用很多。
2. Comparator接口对构成有效实例的内容有很强的要求，有效实例包含其一般合同。通过实施接口，您承诺遵守其合同。
3. 接口配备了大量有用的默认方法来转换和组合比较器。

如果您需要一个与Comparator共享以下一个或多个特性的函数接口，您应该认真考虑编写专用的函数接口而不是使用标准接口：
•它将被普遍使用，并可从描述性名称中受益。
•它与之相关的合同很强。
•它将受益于自定义默认方法。

如果您选择编写自己的函数接口，请记住它是一个接口，因此应该非常谨慎地设计（第21项）。

请注意，EldestEntryRemovalFunction接口（第199页）标有@FunctionalInterface注释。
此注释类型在精神上与@Override类似。
它是程序员意图的声明，有三个目的：
它告诉读者该类及其文档，该接口旨在启用lambdas;
它保持诚实，因为除非它只有一个抽象方法，否则接口不会编译;
并且它可以防止维护者在接口发生时意外地将抽象方法添加到接口。

**始终使用@FunctionalInterface注释注释您的函数接口。**

关于API中函数接口的使用，应该最后一点。
如果可能在客户端中产生可能的歧义，则不要提供具有多个重载的方法，这些方法在同一参数位置采用不同的函数接口。
这不仅仅是一个理论问题。
 ExecutorService的submit方法可以采用Callable <T>或Runnable，并且可以编写一个客户端程序，需要使用强制转换来指示正确的重载（第52项）。
 避免此问题的最简单方法是不要编写在同一参数位置使用不同函数接口的重载。
 这是第52项建议中的一个特例，“明智地使用重载”。

总之，既然Java已经有了lambdas，那么你必须考虑使用lambda来设计你的API。
接受输入上的函数接口类型并在输出上返回它们。通常最好使用java.util.function.Function中提供的标准接口，但请注意相对罕见的情况，即最好编写自己的函数接口。



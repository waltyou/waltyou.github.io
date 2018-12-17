---
layout: post
title: 《Effective Java》学习日志（七）55：谨慎地返回Optional
date: 2018-12-17 19:19:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---


<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---

## 目录
{:.no_toc}

* 目录
{:toc}
---

在Java 8之前，在编写在某些情况下无法返回值的方法时，可以采用两种方法。
您可以抛出异常，也可以返回null（假设返回类型是对象引用类型）。
这些方法都不完美。
应该为异常条件保留异常（第69项），抛出异常是很昂贵的，因为在创建异常时会捕获整个堆栈跟踪。

返回null没有这些缺点，但它有自己的缺点。
如果方法返回null，则客户端必须包含特殊情况代码以处理返回null的可能性，除非程序员可以证明无法返回null。
如果客户端忽略检查空返回并在某些数据结构中存储空返回值，则NullPointerException可能在将来的某个任意时间导致，在代码中与该问题无关的某个位置。

在Java 8中，有第三种方法来编写可能无法返回值的方法。 
Optional<T>类表示一个不可变容器，它可以包含单个非空T引用，也可以不包含任何内容。
一个包含任何内容的Optional被认为是空的。
据说一个值存在于非空的Optional中。
可选的本质上是一个不可变的集合，最多可以容纳一个元素。
Optional<T>没有实现Collection <T>，但原则上是可以的。

概念上，正常情况下返回T，但在某些情况下，可能无法执行此操作的方法，可以声明为返回 Optional<T>。
这允许方法返回空结果以指示它无法返回有效结果。
返回Optional的方法比抛出异常的方法更灵活，更易于使用，并且比返回null的方法更不容易出错。

在第30项中，我们展示了这种方法，根据元素的自然顺序计算集合中的最大值。

    // Returns maximum value in collection - throws exception if empty
    public static <E extends Comparable<E>> E max(Collection<E> c) {
        if (c.isEmpty())
            throw new IllegalArgumentException("Empty collection");
        E result = null;
        for (E e : c)
            if (result == null || e.compareTo(result) > 0)
                result = Objects.requireNonNull(e);
        return result;
    }

如果给定集合为空，则此方法抛出IllegalArgumentException。 
我们在第30项中提到，更好的选择是返回Optional <E>。 以下是修改它时方法的外观：

    // Returns maximum value in collection as an Optional<E>
    public static <E extends Comparable<E>>
            Optional<E> max(Collection<E> c) {
        if (c.isEmpty())
            return Optional.empty();
        E result = null;
        for (E e : c)
            if (result == null || e.compareTo(result) > 0)
                result = Objects.requireNonNull(e);
        return Optional.of(result);
    }


如您所见，返回Optional很简单。 
您所要做的就是使用适当的静态工厂创建Optional。 

在这个程序中，我们使用两个：Optional.empty（）返回一个空的Optional，Optional.of（value）返回一个包含给定非null值的Optional。 
将null传递给Optional.of（value）是一个编程错误。 
如果这样做，该方法通过抛出NullPointerException来响应。 
Optional.ofNullable（value）方法接受一个可能为null的值，如果传入null则返回一个空的Optional。
**永远不要从 Optional-returning 的方法返回一个null值：它会破坏工具的整个目的。**

流上的许多终端操作返回选项。 
如果我们重写max方法来使用流，Stream的max操作会为我们生成一个Optional（尽管我们必须传入一个显式比较器）：

    // Returns max val in collection as Optional<E> - uses stream
    public static <E extends Comparable<E>>
            Optional<E> max(Collection<E> c) {
        return c.stream().max(Comparator.naturalOrder());
    }


那么如何选择返回一个Optional而不是返回一个null或抛出一个异常呢？ 
Optionals在精神上类似于检查异常（第71项），因为它们迫使API的用户面对可能没有返回值的事实。 
抛出未经检查的异常或返回null允许用户忽略此可能性，并可能产生可怕的后果。 
但是，抛出已检查的异常需要在客户端中添加额外的样板代码。

如果方法返回一个Optional，则客户端可以选择在方法无法返回值时要采取的操作。 您可以指定默认值：

    // Using an optional to provide a chosen default value
    String lastWordInLexicon = max(words).orElse("No words...");

或者您可以抛出任何适当的异常。 
请注意，我们传入异常工厂而不是实际异常。 
除非实际抛出异常，否则这将避免创建异常的开销：

    // Using an optional to throw a chosen exception
    Toy myToy = max(toys).orElseThrow(TemperTantrumException::new);

如果你可以证明一个optional是非空的，那么你可以从Optional中获取值，而不指定当optional是空的时要采取的操作，但如果你错了，你的代码将抛出NoSuchElementException：

    // Using optional when you know there’s a return value
    Element lastNobleGas = max(Elements.NOBLE_GASES).get();

有时您可能会遇到这样的情况：获取默认值很昂贵，并且您希望避免这种成本，除非有必要。
对于这些情况，Optional提供了一种方法，该方法接受Supplier <T>并仅在必要时调用它。
这个方法叫做orElseGet，但也许它应该被称为orElseCompute，因为它与名称以compute开头的三个Map方法密切相关。
有几种可选方法可用于处理更专业的用例：filter，map，flatMap和ifPresent。

在Java 9中，添加了另外两个方法：或者ifPresentOrElse。
如果上述基本方法与您的用例不相符，请查看这些更高级方法的文档，看看它们是否能完成这项工作。

如果这些方法都不符合您的需求，Optional提供了isPresent（）方法，可以将其视为安全阀。
如果Optional包含值，则返回true;如果为空，则返回false。您可以使用此方法在可选结果上执行您喜欢的任何处理，但请确保明智地使用它。 
isPresent的许多用途可以有利地被上面提到的方法之一取代。生成的代码通常更短，更清晰，更具惯用性。

例如，请考虑此代码段，它打印进程父进程的进程ID，如果进程没有父进程，则为N/A. 
该代码段使用Java 9中引入的ProcessHandle类：

    Optional<ProcessHandle> parentProcess = ph.parent();
    System.out.println("Parent PID: " + (parentProcess.isPresent() ?
        String.valueOf(parentProcess.get().pid()) : "N/A"));

上面的代码片段可以替换为使用Optional的map函数的代码片段：

    System.out.println("Parent PID: " +
        ph.parent().map(h -> String.valueOf(h.pid())).orElse("N/A"));

使用流进行编程时，发现自己使用Stream <Optional <T >>并要求包含非空选项中的所有元素的Stream <T>以便继续进行并不罕见。
如果你正在使用Java 8，那么这里是如何弥合差距：

    streamOfOptionals
        .filter(Optional::isPresent)
        .map(Optional::get)

在Java 9中，Optional配备了stream（）方法。 
此方法是一个适配器，它将Optional变为包含元素的Stream（如果在Optional中存在，或者如果它为空则为none）。 
结合Stream的flatMap方法（第45项），此方法为上面的代码片段提供了简洁的替代：

    streamOfOptionals.
        .flatMap(Optional::stream)

并非所有返回类型都受益于Optional。
**容器类型（包括集合，映射，流，数组和选项）不应包含在选项中。**
您应该只返回一个空的List <T>（第54项），而不是返回一个空的Optional <List <T >>。
返回空容器将消除客户端代码处理Optional的需要。 
ProcessHandle类确实有arguments方法，它返回Optional <String []>，但是这个方法应该被视为一个不能被模拟的异常。

那么什么时候应该声明一个方法来返回Optional <T>而不是T？
通常，如果可能无法返回结果，则应声明返回Optional <T>的方法，如果未返回结果，则客户端必须执行特殊处理。
也就是说，返回Optional <T>并非没有成本。 
Optional是必须分配和初始化的对象，从Optional中读取值需要额外的间接。
**这使得选项不适合在某些性能关键的情况下使用。特定方法是否属于此类别只能通过仔细测量来确定**（第67项）。

返回包含装箱基元类型的Optional与返回基元类型相比非常昂贵，因为Optional具有两个级别的装箱而不是零。
因此，库设计人员认为适合为基本类型int，long和double提供Optional <T>的类似物。
这些Optional类型是OptionalInt，OptionalLong和OptionalDouble。
它们包含Optional <T>中的大多数但不是全部方法。
**因此，您永远不应该返回它们包含Optional的装箱基元类型**，可能的例外是“次要基元类型”，布尔，字节，字符，短和浮点数。

到目前为止，我们已经讨论了返回选项并在返回后处理它们。
我们还没有讨论其他可能的用途，这是因为Optional的大多数其他用途都是可疑的。
例如，您永远不应该使用选项作为map值。
如果这样做，您有两种方法可以从map中表示键的逻辑缺失：键可以不在map中，也可以存在并映射到空的Optional。
这代表了不必要的复杂性，具有很大的混淆和错误的可能性。
**通常来讲，将Optional用作集合或数组中的键，值或元素几乎从不合适。**

这留下了一个无法回答的大问题。
是否适合在实例字段中存储Optional？通常它是一种“难闻的气味”：它表明也许你应该有一个包含可选字段的子类。
但有时它可能是合理的。
考虑第2项中我们的NutritionFacts类的情况.AdamitionFacts实例包含许多不需要的字段。
对于这些字段的每种可能组合，您都不能拥有子类。
此外，这些字段具有原始类型，这使得直接表达缺席变得尴尬。 
NutritionFacts的最佳API将为每个可选字段从getter返回一个Optional，因此将这些选项作为字段存储在对象中是很有意义的。

总之，如果您发现自己编写的方法无法始终返回值，并且您认为方法的用户每次调用它时都考虑这种可能性，那么您应该返回一个Optional。
但是，您应该意识到返回选项会产生真正的性能后果;对于性能关键的方法，最好返回null或抛出异常。
最后，在其他返回值数量少于1的情况下，应尽量少的使用 optional。





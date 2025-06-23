---
layout: post
title: 《Effective Java》学习日志（十）83:谨慎的延迟初始化
date: 2019-02-11 13:55:00
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

延迟初始化是延迟字段初始化直到需要其值的行为。 如果永远不需要该值，则永远不会初始化该字段。 此技术适用于静态和实例字段。 虽然延迟初始化主要是一种优化，但它也可以用于打破类和实例初始化中的有害循环。 

与大多数优化一样，延迟初始化的最佳建议是“除非你需要，否则不要这样做”（第67项）。 懒惰的初始化是一把双刃剑。 它降低了初始化类或创建实例的成本，但代价是增加了访问延迟初始化字段的成本。 取决于这些字段最终需要初始化的部分，初始化它们的成本是多少，以及初始化后每个字段的访问频率，延迟初始化（如许多“优化”）实际上会损害性能。

也就是说，延迟初始化有其用途。 如果仅在类的一小部分实例上访问字段，并且初始化字段的成本很高，则延迟初始化可能是值得的。 确切知道的唯一方法是使用和不使用延迟初始化来测量类的性能。

在存在多个线程的情况下，延迟初始化很棘手。 如果两个或多个线程共享一个延迟初始化的字段，则必须采用某种形式的同步，否则可能导致严重的错误（第78项）。 此项中讨论的所有初始化技术都是线程安全的。

**在大多数情况下，正常初始化优于延迟初始化**。 以下是通常初始化的实例字段的典型声明。 注意使用final修饰符（Item 17）：

```java
// Normal initialization of an instance field
private final FieldType field = computeFieldValue();
```

**如果使用延迟初始化来破坏初始化循环，请使用同步访问器**，因为它是最简单，最清晰的替代方法：

```java
// Lazy initialization of instance field - synchronized accessor
private FieldType field;

private synchronized FieldType getField() {
    if (field == null)
    	field = computeFieldValue();
    return field;
}
```

当应用于静态字段时，这两个习惯用法（正常初始化和使用同步访问器的延迟初始化）都不会更改，除了您将static修饰符添加到字段和访问器声明。

**如果需要在静态字段上使用延迟初始化来提高性能，请使用延迟初始化持有者类习惯用法**。 这个成语利用了在使用类之前不会初始化类的保证。 以下是它的外观：

```java
// Lazy initialization holder class idiom for static fields
private static class FieldHolder {
	static final FieldType field = computeFieldValue();
}

private static FieldType getField() { return FieldHolder.field; }
```

当第一次调用getField时，它首次读取FieldHolder.field，导致FieldHolder类的初始化。 这个习惯用法的优点在于getField方法不是同步的，只执行字段访问，因此延迟初始化几乎不会增加访问成本。 典型的VM将仅同步字段访问以初始化类。 初始化类后，VM会对代码进行修补，以便后续访问该字段不涉及任何测试或同步。

**如果需要在实例字段上使用延迟初始化来提高性能，请使用双重检查惯用法**。 这个习惯用法避免了初始化后访问字段时锁定的成本（第79项）。 成语背后的想法是检查字段的值两次（因此名称仔细检查）：一次没有锁定，然后，如果字段看起来是未初始化的，则第二次锁定。 仅当第二次检查表明该字段未初始化时，该呼叫才会初始化该字段。 因为字段初始化后没有锁定，所以将字段声明为volatile是非常重要的（第78项）。 这是成语：

```java
// Double-check idiom for lazy initialization of instance fields
private volatile FieldType field;

private FieldType getField() {
    FieldType result = field;
    if (result == null) { // First check (no locking)
        synchronized(this) {
            if (field == null) // Second check (with locking)
            	field = result = computeFieldValue();
        }
    }
    return result;
}
```

此代码可能看起来有点复杂。 特别是，对局部变量（结果）的需求可能不清楚。 这个变量的作用是确保该字段在已经初始化的常见情况下只读一次。 虽然不是绝对必要，但这可以提高性能，并且通过应用于低级并发编程的标准更加优雅。 在我的机器上，上面的方法大约是没有局部变量的明显版本的1.4倍。

虽然您也可以将双重检查成语应用于静态字段，但没有理由这样做：延迟初始化持有者类习惯用法是更好的选择。

双重检查成语的两个变种注意到。 有时，您可能需要懒惰地初始化一个可以容忍重复初始化的实例字段。 如果你发现自己处于这种情况，你可以使用复核的双重检查成语，省去第二次检查。 毫不奇怪，它被称为单一检查成语。 这是它的外观。 请注意，该字段仍然声明为 *volatile*：

```java
// Single-check idiom - can cause repeated initialization!
private volatile FieldType field;

private FieldType getField() {
    FieldType result = field;
    if (result == null)
        field = result = computeFieldValue();
    return result;
}
```

本项中讨论的所有初始化技术都适用于原始字段以及对象引用字段。 当将双重检查或单一检查惯用法应用于数字原始字段时，将针对0（数字原始变量的默认值）而不是空来检查字段的值。

如果你不关心每个线程是否重新计算字段的值，并且字段的类型是long或double以外的原语，那么你可以选择从单一检查成语中的字段声明中删除volatile修饰符。 这种变体被称为生动的单一检查成语。 它加速了某些体系结构上的字段访问，但代价是额外的初始化（每个访问该字段的线程最多一个）。 这绝对是一种奇特的技术，不适合日常使用。

总之，您应该正常初始化大多数字段，而不是懒惰。 如果必须懒惰地初始化字段以实现性能目标或打破有害的初始化循环，则使用适当的延迟初始化技术。 例如字段，它是双重检查成语; 对于静态字段，惰性初始化持有者类成语。 例如，可以容忍重复初始化的字段，您也可以考虑单一检查习语。

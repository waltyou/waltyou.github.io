---
layout: post
title: 《Effective Java》学习日志（十一）90:使用序列化代理替代序列化实例
date: 2019-02-19 17:17:00
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

如第85和第86项所述并在本章中讨论过，实现Serializable的决定增加了错误和安全问题的可能性，因为它允许使用extralinguistic机制代替普通构造函数创建实例。 然而，有一种技术可以大大降低这些风险。 此技术称为序列化代理模式（serialization proxy pattern）。

序列化代理模式相当简单。 首先，设计一个私有静态嵌套类，它简洁地表示封闭类的实例的逻辑状态。 这个嵌套类称为封闭类的序列化代理。 它应该有一个构造函数，其参数类型是封闭类。 此构造函数仅复制其参数中的数据：它不需要执行任何一致性检查或防御性复制。 根据设计，序列化代理的默认序列化形式是封闭类的完美序列化形式。 必须声明封闭类及其序列化代理以实现Serializable。

例如，考虑在Item 50中编写的不可变Period类，并在Item 88中进行序列化。这是该类的序列化代理。 Period非常简单，其序列化代理与该字段具有完全相同的字段：

```java
// Serialization proxy for Period class
private static class SerializationProxy implements Serializable {
    private final Date start;
    private final Date end;
    SerializationProxy(Period p) {
        this.start = p.start;
        this.end = p.end;
    }
    private static final long serialVersionUID =
    			234098243823485285L; // Any number will do (Item 87)
}
```

接下来，将以下**writeReplace**方法添加到封闭类中。 可以将此方法逐字复制到具有序列化代理的任何类中：

```java
// writeReplace method for the serialization proxy pattern
private Object writeReplace() {
	return new SerializationProxy(this);
}
```

封闭类上存在此方法会导致序列化系统发出SerializationProxy实例而不是封闭类的实例。 换句话说，writeReplace方法在序列化之前将封闭类的实例转换为其序列化代理。

使用此writeReplace方法，序列化系统将永远不会生成封闭类的序列化实例，但攻击者可能会构造一个试图违反类不变量的实例。 要确保此类攻击失败，只需将此readObject方法添加到封闭类：

```java
// readObject method for the serialization proxy pattern
private void readObject(ObjectInputStream stream)
    		throws InvalidObjectException {
    throw new InvalidObjectException("Proxy required");
}
```

最后，在SerializationProxy类上提供readResolve方法，该方法返回封闭类的逻辑等效实例。 此方法的存在导致序列化系统在反序列化时将序列化代理转换回封闭类的实例。

这个readResolve方法只使用它的公共API创建一个封闭类的实例，其中就是模式的美妙之处。 它在很大程度上消除了序列化的语言特征，因为反序列化的实例是使用与任何其他实例相同的构造函数，静态工厂和方法创建的。 这使您不必单独确保反序列化的实例服从类的不变量。 如果类的静态工厂或构造函数建立这些不变量并且其实例方法维护它们，那么您已确保不变量也将通过序列化来维护。

以下是Period.SerializationProxy的readResolve方法：

```java
// readResolve method for Period.SerializationProxy
private Object readResolve() {
	return new Period(start, end); // Uses public constructor
}
```

与防御性复制方法（第357页）一样，序列化代理方法可以阻止伪造的字节流攻击（第354页）和内部字段盗窃攻击（第356页）。 与前两种方法不同，这一方法允许Period的字段为final，这是Period类真正不可变所必需的（第17项）。 与之前的两种方法不同，这一方法不涉及很多想法。 您不必弄清楚哪些字段可能会被狡猾的序列化攻击所破坏，也不必在反序列化过程中明确执行有效性检查。

还有另一种方法，序列化代理模式比readObject中的防御性复制更强大。 序列化代理模式允许反序列化实例具有与最初序列化实例不同的类。 您可能不认为这在实践中有用，但确实如此。

考虑EnumSet的情况（第36项）。 这个类没有公共构造函数，只有静态工厂。 从客户端的角度来看，它们返回EnumSet实例，但在当前的OpenJDK实现中，它们返回两个子类中的一个，具体取决于底层枚举类型的大小。 如果底层枚举类型包含64个或更少的元素，则静态工厂返回RegularEnumSet; 否则，他们返回一个JumboEnumSet。

现在考虑如果序列化其枚举类型具有六十个元素的枚举集，然后再向枚举类型添加五个元素，然后反序列化枚举集，会发生什么。 它是序列化时的RegularEnumSet实例，但最好是反序列化后的JumboEnumSet实例。 实际上，这正是发生的事情，因为EnumSet使用序列化代理模式。 如果你很好奇，这里是EnumSet的序列化代理。 这真的很简单：

```java
// EnumSet's serialization proxy
private static class SerializationProxy <E extends Enum<E>>
    		implements Serializable {
    // The element type of this enum set.
    private final Class<E> elementType;
    // The elements contained in this enum set.
    private final Enum<?>[] elements;
    SerializationProxy(EnumSet<E> set) {
        elementType = set.elementType;
        elements = set.toArray(new Enum<?>[0]);
    }
    private Object readResolve() {
        EnumSet<E> result = EnumSet.noneOf(elementType);
        for (Enum<?> e : elements)
        	result.add((E)e);
        return result;
    }
    private static final long serialVersionUID = 362491234563181265L;
}
```

序列化代理模式有两个限制。 它与用户可扩展的类不兼容（第19项）。 此外，它与某些对象图包含圆形的类不兼容：如果您尝试从其序列化代理的readResolve方法中调用此类对象上的方法，您将获得ClassCastException，因为您还没有该对象， 只有它的序列化代理。

最后，序列化代理模式的附加功能和安全性不是免费的。 在我的机器上，使用序列化代理序列化和反序列化Period实例比使用防御性复制更加昂贵14％。

总之，只要您发现自己必须在不能由其客户端扩展的类上编写readObject或writeObject方法，请考虑序列化代理模式。 这种模式可能是使用非平凡不变量强健序列化对象的最简单方法。
